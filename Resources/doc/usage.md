Usage
-----

CoopTilleulsForgotPasswordBundle provides 2 events allowing you to build your own business:
- `coop_tilleuls_forgot_password.create_token`: dispatched when user requests a new password (`POST /forgot_password/`)
- `coop_tilleuls_forgot_password.update_password`: dispatched when user has reset its password (`POST /forgot_password/{token}`)

## Send email on user request

On the first user story, user will send its identifier (email, username...), you'll have to send a custom email
allowing him to reset its password using a valid PasswordToken.

```php
// src/EventSubscriber/ForgotPasswordEventSubscriber.php
namespace App\EventSubscriber;

// ...
use CoopTilleuls\ForgotPasswordBundle\Event\CreateTokenEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;
use Twig\Environment;

final class ForgotPasswordEventSubscriber implements EventSubscriberInterface
{
    private $mailer;
    private $twig;

    public function __construct(MailerInterface $mailer, Environment $twig)
    {
        $this->mailer = $mailer;
        $this->twig = $twig;
    }

    public static function getSubscribedEvents()
    {
        return [
            // Symfony 4.3 and inferior, use 'coop_tilleuls_forgot_password.create_token' event name
            CreateTokenEvent::class => 'onCreateToken',
        ];
    }

    public function onCreateToken(CreateTokenEvent $event)
    {
        $passwordToken = $event->getPasswordToken();
        $user = $passwordToken->getUser();

        $message = (new Email())
            ->from('no-reply@example.com')
            ->to($user->getEmail())
            ->subject('Reset your password')
            ->html($this->twig->render(
                'App:ResetPassword:mail.html.twig',
                [
                    'reset_password_url' => sprintf('http://www.example.com/forgot-password/%s', $passwordToken->getToken()),
                ]
            ));
        if (0 === $this->mailer->send($message)) {
            throw new \RuntimeException('Unable to send email');
        }
    }
}
```

Your app is ready to receive a JSON request like:

```json
{
    "email": "foo@example.com"
}
```

## Update the password of the user

On the second user story, user will send its new password, and you'll have to encode it and save it.

```php
// src/EventSubscriber/ForgotPasswordEventSubscriber.php
namespace App\EventSubscriber;

// ...
use CoopTilleuls\ForgotPasswordBundle\Event\UpdatePasswordEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class ForgotPasswordEventSubscriber implements EventSubscriberInterface
{
    private $userManager;

    public function __construct(UserManager $userManager)
    {
        $this->userManager = $userManager;
    }

    public static function getSubscribedEvents()
    {
        return [
            // Symfony 4.3 and inferior, use 'coop_tilleuls_forgot_password.update_password' event name
            UpdatePasswordEvent::class => 'onUpdatePassword',
        ];
    }

    public function onUpdatePassword(UpdatePasswordEvent $event)
    {
        $passwordToken = $event->getPasswordToken();
        $user = $passwordToken->getUser();
        $user->setPlainPassword($event->getPassword());
        $this->userManager->updateUser($user);
    }
}
```

Your app is ready to receive a request like:

```json
{
    "password": "P4$$w0rd"
}
```
