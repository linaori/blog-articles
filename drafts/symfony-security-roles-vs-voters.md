[//]: # (TITLE: Symfony Security Roles vs. Voters)
[//]: # (DATE: 2016-08-20T09:00:00+01:00)
[//]: # (TAGS: php, security, roles, voters, authentication, authorization, firewall, access control)

In my [previous blog bost][1] I've explained the basics of authentication, authorization and how this is dealt with in
Symfony. Due to the size of the post, I've left out several important topics such as roles and voters; Both an equally
important part of authentication and authorization. A common misconception is that roles should be used to check 
permissions. In fact, they should definitely not be used to check permissions! 

## Roles and Authentication

Roles are primarily for authentication as they extend on the part of identification. A role describes something about a
user, for example `ROLE_USER` defines I'm a normal user and `ROLE_ADMIN` could define that I'm an administrator. In the
[Security documentation][2] it's explained how the `ROLE_` prefix is used and how this fits in with authorization. It
explains how the `ROLE_USER` is commonly assigned and how to check this for access with `access_control`. It also
briefly mentions the role hierarchy and how this is used to vote on dynamic roles; E.g. if you've got `ROLE_ADMIN` you
can have it virtually assign the `ROLE_USER` automatically.

While the role hierarchy looks interesting, it has nothing to do with authentication. In fact, this is the authorization
dealing with this virtual inheritance. The only way to trigger this, is by checking if you're allowed to do something;
Authorization. The example is pointing at `access_control` verifying if you have the required role for a specific route.
While this may seem nice, this is not how you should be used `access_control`.

## Voters and Authorization

So what should you be using then? Voters. Voters are classes that simply vote on an attribute and optionally a subject.
An attribute is usually an uppercase string that defines an action. A subject is the subject being voted on if required.
Did you know that the only reason you can vote on (dynamic) roles, is because of the [RoleVoter][3] and 
[RoleHierarchyVoter][4]? They simply check if the token contains the roles specified. 

The [symfony documentation explains Authorization][5] if you want to dive a bit deeper into its inner workings. 
[Voters][6] basically come down to the following:
 - Can I vote on this attribute?
 - When I vote on this attribute do I return true or false?

Voters are triggered for every authorization part:
 - The `access_control` configuration triggers them;
 - The `@Security` annotation triggers them;
 - The `AuthorizationChecker` uses it via the `AccessDecisionManager`.
 
All of the above authorization methods use an attribute (or multiple) and a subject to vote on.

### So Why Should I Use Voters Instead of Roles?

As I've explained, roles are merely an extension to authentication, they serve as extra descriptions to your identity.
calling something like `$authorizationChecker->isGranted('ROLE_ADMIN')` doesn't really make sense, what are you actually
checking here? Let's say that I have a button to edit a forum post:
 - The owner may edit it;
 - The admin may edit it;
 - A moderator may edit it.

Let's add the link to the edit page:
```twig
{% if post.owner.id is app.user.username or is_granted('ROLE_MODERATOR') or is_granted('ROLE_ADMIN') %}
    <a href="{{ path('...') }}">Edit Post</a>
{% endif %}
```
And let's add the permission check in the controller:

```php
public function editPostAction(Post $post)
{
    // ... 
    /** @var $token \Symfony\Component\Security\Core\Authentication\Token\TokenInterface */
    /** @var $AuthorizationChecker \Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface */
    if ($post->getOwner()->getId() !== $token->getUsername()
        && !$AuthorizationChecker->isGranted('ROLE_MODERATOR')
        && !$AuthorizationChecker->isGranted('ROLE_ADMIN')
    )) {
        throw new AccessDeniedHttpException();
    }
    // ...
}
```

As you can see, this is quite some logic just to check if the current user can see it. Now we want to add another
condition; The post may not be locked. Lets do update the template!

```twig
{% if (post.owner.id is app.user.username and not post.locked)
    or is_granted('ROLE_MODERATOR') 
    or is_granted('ROLE_ADMIN')
%}
    <a href="{{ path('...') }}">Edit Post</a>
{% endif %}
```
Done, right? Oh, we still need to update the controller as well.

```php
<?php

public function editPostAction(Post $post)
{
    // ... 
    /** @var $token \Symfony\Component\Security\Core\Authentication\Token\TokenInterface */
    /** @var $AuthorizationChecker \Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface */
    if (($post->getOwner()->getId() !== $token->getUsername() || $post->isLocked())
        && !$AuthorizationChecker->isGranted('ROLE_MODERATOR')
        && !$AuthorizationChecker->isGranted('ROLE_ADMIN')
    ) {
        throw new AccessDeniedHttpException();
    }
    // ...
}
```
All set, `git push` and be done with it. Except that you product owner wants this link shown in the topic overview as
well as in the post itself. Well, that's going to be a big copy paste... So how can you improve this?

### Creating a Voter

The solution is rather simple, create a voter. The easiest way to create a voter is by [extending the `Voter`][7] that
Symfony provides already. There's a view things we need to decide before making the class:
 - What will it vote on, or the attribute, what should it be called?
 - Do we have a subject or not?
 - What would give it access?

First off we start by making a class:

```php
<?php
namespace App\Security\Voter;

use App\Entity\Post;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class EditPostVoter extends Voter
{    
    protected function supports($attribute, $subject)
    {
        // we only want to vote if the attribute and subject are what we expect
        return $attribute === 'CAN_EDIT_POST' && $subject instanceof Post;
    }
    
    protected function voteOnAttribute($attribute, $subject, TokenInterface $token)
    {
        // our previous business logic indicates that mods and admins can do it regardless
        foreach ($token->getRoles() as $role) {
            if (in_array($role, ['ROLE_MODERATOR', 'ROLE_ADMIN'])) {
                return true;
            }
        }   
        
        /** @var $subject Post */
        return $subject->getOwner()->getId() === $token->getUsername() && !$subject->isLocked();
    }
}
````
> You can also use the [role hierarchy with the access decision manager][8] if you want virtual roles.

The next thing to do, is create a service definition so the security picks it up. It's as simple as adding a tag.

```yml
# app/config/services.yml
services:
    app.security.voter.edit_post:
        class: App\Security\Voter\EditPostVoter
        tags:
            - { name: security.voter }
```

The last things are to replace the security checks.

```php
<?php
// controller
public function editPostAction(Post $post)
{
    /** @var $AuthorizationChecker \Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface */
    if (!$AuthorizationChecker->isGranted('CAN_EDIT_POST', $post)) {
        throw new AccessDeniedHttpException();
    }
    
    // ...
}
```

```yml
{% if is_granted('CAN_EDIT_POST', post) %}
    <a href="{{ path('...') }}">Edit Post</a>
{% endif %}
```

[1]: ./the-basics-of-symfony-security
[2]: http://symfony.com/doc/current/security.html#denying-access-roles-and-other-authorization
[3]: https://github.com/symfony/symfony/blob/master/src/Symfony/Component/Security/Core/Authorization/Voter/RoleVoter.php
[4]: https://github.com/symfony/symfony/blob/master/src/Symfony/Component/Security/Core/Authorization/Voter/RoleHierarchyVoter.php
[5]: http://symfony.com/doc/current/components/security/authorization.html
[6]: http://symfony.com/doc/current/security/voters.html
[7]: http://symfony.com/doc/current/security/voters.html#the-voter-interface
[8]: http://symfony.com/doc/current/security/voters.html#checking-for-roles-inside-a-voter