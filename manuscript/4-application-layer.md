# Application Layer

We continue our example.
The application is growing, and new fields have been added to the registration form: date of birth and an option for agreeing to receive email newsletters.

```php
public function store(
    Request $request, 
    ImageUploader $imageUploader) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    $avatarFileName = ...;    
    $imageUploader->upload(
        $avatarFileName, $request->file('avatar'));
        
    $user = new User();
    $user->email = $request['email'];
    $user->name = $request['name'];
    $user->avatarUrl = $avatarFileName;
    $user->subscribed = $request->has('subscribed');
    $user->birthDate = new DateTime($request['birthDate']);
    
    if(!$user->save()) {
        return redirect()->back()->withMessage('...');
    }
    
    \Email::send($user->email, 'Hi email');
        
    return redirect()->route('users');
}
```
Then, the application gets an API for the mobile app, and user registration must also be implemented there.
Let's also imagine some console artisan command that imports users and wants to register them.
And a Facebook bot!
As a result, the application has several interfaces in addition to the standard HTML, and you need to use actions like user registration or article publication everywhere.
The most natural solution here is to extract the common logic of working with an entity (User in this example) into a separate class.
Such classes are often called service classes:

```php
final class UserService
{
    public function getById(...): User;
    public function getByEmail(...): User;

    public function create(...);
    public function ban(...);
    ...
}
```
However, many application interfaces (API, Web, etc.) are only one of the reasons for creating service classes.
Controller methods start to grow and usually contain two different logics:

```php
public function doSomething(Request $request, $id)
{
    $entity = Entity::find($id);
    
    if (!$entity) {
        abort(404);
    }
    
    if (count($request['options']) < 2) {
        return redirect()->back()->withMessage('...');
    }
    
    if ($entity->something) {
        return redirect()->back()->withMessage('...');
    }
    
    \Db::transaction(function () use ($request, $entity) {
        $entity->someProperty = $request['someProperty'];
        
        foreach($request['options'] as $option) {
            //...
        }
        
        $entity->save();
    });
    
    return redirect()->...
}
```
This method implements at least two responsibilities: the logic for handling HTTP requests/responses and the business logic. Each time developers change the HTTP logic, they are forced to read a lot of business logic code and vice versa. Such code is harder to debug and refactor, so moving the logic into service classes can also be a good idea for this project.

## Passing Request Data

Let's start by creating the **UserService** class.
The first challenge will be passing request data to it.
Some methods don't need much data; for example, only its ID is required to delete an article. 
However, for actions like user registration, a lot of data may be needed. 
We can't use the **Request** class because it's only available for the web and not accessible, 
for instance, in the console. Let's try simple arrays:

```php
final class UserService
{
    public function __construct(
        private ImageUploader $imageUploader, 
        private EmailSender $emailSender) {}
    
    public function create(array $request)
    {
        $avatarFileName = ...;
        $this->imageUploader->upload(
            $avatarFileName, $request['avatar']);

        $user = new User();
        $user->email = $request['email'];
        $user->name = $request['name'];
        $user->avatarUrl = $avatarFileName;
        $user->subscribed = isset($request['subscribed']);
        $user->birthDate = new DateTime($request['birthDate']);
        
        if (!$user->save()) {
            return false;
        }
        
        $this->emailSender->send($user->email, 'Hi email');
        
        return true;
    }
}

// Controller
public function store(Request $request, UserService $userService) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    if (!$userService->create($request->all())) {
        return redirect()->back()->withMessage('...');
    }
    
    return redirect()->route('users');
}
```
I've extracted the logic "as is" and see the issue. When we try to register a user from the console, the code will look something like this:

```php
$data = [
    'email' => $email,
    'name' => $name,
    'avatar' => $avatarFile,
    'birthDate' => $birthDate->format('Y-m-d'),
];

if ($subscribed) {
    $data['subscribed'] = true;
}

$userService->create($data);
```
It looks overly convoluted.
The HTML-specific data structure has also moved into the service with the user creation code.
Boolean values are checked by the presence of the key in the array. Date values are obtained by converting from strings.
Using such a data format is quite inconvenient if this data doesn't come from an HTML form.
We need a new data format, something more natural and user-friendly.
The **Data Transfer Object**(DTO) pattern suggests simply creating objects with the required fields:

```php
final class UserCreateDto
{
    private string $email;
    
    private DateTime $birthDate;
    
    private bool $subscribed;
    
    public function __construct(
        string $email, DateTime $birthDate, bool $subscribed) 
    {
        $this->email = $email;
        $this->birthDate = $birthDate;
        $this->subscribed = $subscribed;
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
        
    public function getBirthDate(): DateTime 
    {
        return $this->birthDate;
    }
    
    public function isSubscribed(): bool
    {
        return $this->subscribed;
    }
}
```

I often hear objections like, "I don't want to create a whole class just to pass data. Arrays can do the job just as well." This is partially true, and creating DTO classes may not be necessary for a certain level of application complexity.

In the book, I will provide a couple of arguments in favor of DTOs. Here, I'll just mention that in modern IDEs, creating such a class is very fast - you only need to define the fields, and constructor parameters and getter methods are generated automatically.

Developers will rarely need to inspect this class, so it doesn't add much complexity to application maintenance. With the introduction of read-only fields and classes in modern PHP, creating such DTO classes has become even easier.

```php
readonly final class UserCreateDto
{   
    public function __construct(
        public string $email;
        public DateTime $birthDate;
        public bool $subscribed;
    ) {}
}
```

The combination of a private field and a getter method was used to ensure data immutability within the DTO object. The `readonly` modifier provides immutability for the object, so we can make the fields public.

```php
final class UserService
{
    //...
    
    public function create(UserCreateDto $request)
    {
        $avatarFileName = ...;
        $this->imageUploader->upload(
            $avatarFileName, $request->avatarFile);

        $user = new User();
        $user->email = $request->email;
        $user->avatarUrl = $avatarFileName;
        $user->subscribed = $request->subscribed;
        $user->birthDate = $request->birthDate;
        
        if (!$user->save()) {
            return false;
        }
        
        $this->emailSender->send($user->email, 'Hi email');
        
        return true;
    }
}

public function store(Request $request, UserService $userService) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    $dto = new UserCreateDto(
        $request['email'], 
        new DateTime($request['birthDate']), 
        $request->has('subscribed'));
    
    if (!$userService->create($dto)) {
        return redirect()->back()->withMessage('...');
    }
    
    return redirect()->route('users');
}
```
Now, it looks canonical. The service class receives a clean DTO and acts. However, the controller method is now quite large, and the constructor of the DTO class can be lengthy. You can move the logic for handling request data out of there. Laravel has convenient classes for this - **Form Requests**.

```php
final class UserCreateRequest extends FormRequest
{
    public function rules()
    {
        return [
            'email' => 'required|email',
            'name' => 'required',
            'avatar' => 'required|image',
            'birthDate' => 'required|date',
        ];
    }
    
    public function getDto(): UserCreateDto
    {
        return new UserCreateDto(
            $this->get('email'), 
            new DateTime($this->get('birthDate')), 
            $this->has('subscribed'));
    }
}

final class UserController extends Controller
{
    public function store(
        UserCreateRequest $request, UserService $userService) 
    {        
        if (!$userService->create($request->getDto())) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('users');
    }
}
```
If any class requests a class that inherits from **FormRequest** as a dependency, Laravel will create it and automatically perform validation. In the case of incorrect data, the **store** method will not be executed, so you can always be confident in the validity of the data in **UserCreateRequest**.

## Working with database

Simple example:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        $post = Post::find($id);
        
        if (!$post) {
            abort(404);
        }
        
        if (!$postService->publish($post)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}

final class PostService
{
    public function publish(Post $post)
    {
        $post->published = true;

        return $post->save();
    }
}
```
Publishing a post is an example of the simplest non-CRUD action, and I'll use it often.
The example looks good, however, if we try to call the service from a console command, we must fetch the **Post** entity from the database again.

```php
public function handle(PostService $postService)
{
    $post = Post::find(...);
    
    if (!$post) {
        $this->error(...);
        return;
    }
    
    if (!$postService->publish($post)) {
        $this->error(...);
    } else {
        $this->info(...);
    }
}
```

This is an example of Single Responsibility Principle (SRP) violation.
Each part of the application(service classes, web controllers, console commands) works with database.
Each change, related to database, might cause changes in whole application.
Working with database should be encapsulated in one place, and the service class is a good candidate.
Sometimes, I see an interesting way of encapsulation by using **getById** method:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        $post = $postService->getById($id);
        
        if (!$post) {
            abort(404);
        }
        
        if (!$postService->publish($post)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}
```

This controller method just gets the post entity and provides it to the `publish` method. 
There is no need to have it in this method. The more simple and natural solution is:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        if (!$postService->publish($id)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}

final class PostService
{
    public function publish(int $id)
    {
        $post = Post::find($id);
            
        if (!$post) {
            return false;
        }
        
        $post->published = true;
        
        return $post->save();
    }
}
```
One of the main advantages of creating service classes is consolidating all work with business logic and infrastructure, including working with data storage, such as databases and files, in one place, leaving Web, API, Console, and other interfaces to handle only their responsibilities. The web part (web controllers) should prepare data for the service classes and display the results to the user. The same applies to other interfaces. This is the Single Responsibility Principle for layers.
Layers? Yes.

A layer is a group of classes united by similar responsibilities and dependencies. For example, the layer of web controllers - classes that accept a web request, pass it to the service classes, receive a response, and return the result in the required form. They are called layers because the web request (or console request) passes through these layers (of web controllers, service classes, database operations, etc.) and returns the result through them as well.

All service classes, hiding the application's logic within themselves, form a structure that has many names:
* **Service layer**, because of the **service** classes.
* **Application layer**, because it contains all the application logic, excluding interfaces (web controllers, console commands, etc.).
* In GRASP, this layer is called **Controllers layer**, since the service classes there are called controllers.
* There are likely other names as well.

In this book, I will call it the **Application layer**.

What I described here is very similar to the architectural pattern **Hexagonal architecture**, **Onion architecture**, or dozens of similar ones. Unsurprisingly, since they solve exactly the same problems. However, it's much more beneficial for a developer's growth to understand the reasons for separating code into different classes on their own, and to feel which parts of the code can work with the database or web request data. To see how sections of code with similar responsibilities and needs are arranged into something that can be called layers. Having gained such experience and honed their skills, developers can familiarize themselves with these patterns and, possibly, adopt one of them as a standard for a project, fully understanding that this pattern suits the project, not as a microscope used to drive nails or a cannon fired at sparrows.

## Service classes or command classes?

When an entity is growing, its service class will also become large. Different actions, such as editing or publishing an article, require different dependencies. Everyone needs a database, but some want to send emails, and others wish to upload a file to storage and call some external API. The number of parameters in the service class constructor snowballs, although each may be used in only one or two methods. It quickly becomes clear that the class is doing too many things. Therefore, developers often start creating classes for each action with entities.

As far as I know, there is no standard for naming such classes either. I've seen the suffix **UseCase** (for example, **PublishPostUseCase**), the suffix **Action** (**PublishPostAction**), but I prefer the suffix **Command**: **CreatePostCommand**, **PublishPostCommand**, **DeletePostCommand**.

```php
final class PublishPostCommand
{
    public function execute($id)
    {
        //...
    }
}
```
In the **Command Bus** pattern, the suffix **Command** is used for DTO (Data Transfer Object) classes, while the classes that execute commands are called **CommandHandlers**.

```php
final class ChangeUserPasswordCommand
{
    //...
}

final class ChangeUserPasswordCommandHandler
{
    public function handle(
        ChangeUserPasswordCommand $command)
    {
        //...
    }
}

// or if one class executes many commands

final class UserCommandHandler
{
    public function handleChangePassword(
        ChangeUserPasswordCommand $command)
    {
        //...
    }
}
```

For simplicity, I will use **Service** classes in the book.

A small note about long class names.
"**ChangeUserPasswordCommandHandler** — wow! Isn't that name too long? I don't want to write it every time!" You'll have to write it only once — when creating it. After that, the IDE will suggest it everywhere. A good, fully descriptive class name is much more important. Every developer can immediately say what the class **ChangeUserPasswordCommandHandler** roughly does without looking inside it.

## A few words at the end of the chapter

Isolating the application layer is a very responsible step, and the reason must be serious. There are only two:

1. Different interfaces to the same actions (Web, API, Console, various bots). Here, the extraction of common logic is quite apparent.
2. Expanded and complex logic of handling web requests, for example, and business logic. In this case, separating logic into different places can significantly improve the cohesion of the code. Typically, this leads to increased code robustness.