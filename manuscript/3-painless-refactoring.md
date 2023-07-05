# Painless Refactoring

A> A securely fixed patient does not need anesthesia — folk wisdom.

## "Static" Typing

Small and large software projects differ in many aspects, including their work style. In small projects, most time is spent on actual coding - building the codebase. In large projects, however, it involves navigating through this codebase: moving from one class to another, from a method call to its implementation, from method code to its calls (Find Usages). Sometimes, more than 95% of the time is spent on precisely this code maze wandering.

To make this wandering less painful, projects constantly require refactoring, which includes:
- Extracting methods and classes from other methods and classes.
- Renaming them, adding and removing parameters.

Modern integrated development environments (IDEs) have many tools for advanced navigation and refactoring, making it easier and sometimes even performing it fully automatically. However, the dynamic nature of PHP often throws a wrench into the works.

```php
public function publishPost($id)
{
    $post = Post::find($id);
    $post->publish();
}

// or

public function publishPost($post)
{
    $post->publish();
}
```

In both cases, the IDE cannot understand on its own that the **publish** method of the **Post** class was called. To add a new parameter to this method, the developer would have to find all the usages of this method.

```php
public function publish(User $publishedBy)
```

The IDE won't be able to find them automatically. The developer must search the entire project for "publish" and find the method invocations among the results. This can be quite painful for some more common words (like "name" or "create") and large projects.

Let's imagine a situation where a team discovers that the `email` field in the database contains values that are not valid email addresses. How did this happen? Team members must find all possible assignments to the **email** field of the **User** class and verify them. It's a rather challenging task, especially considering that the `email` field is virtual and can be assigned like this:

```php
$user = User::create($request->all());
//or
$user->fill($request->all());
```

This magic, which helped us quickly create applications, reveals its true nature by presenting such surprises. Such bugs in production can be critical, and every minute matters. I still remember spending an entire day searching for all possible assignments to a single field, trying to find where it was receiving an incorrect value in a huge project (millions of lines of code).

After some similar cases of difficult debugging and complex refactoring, I developed a rule: make PHP code as static as possible. The IDE should know everything about every method and field I use.

```php
public function publish(Post $post)
{
    $post->publish();
}

// or with phpDoc

public function publish($id)
{
    /**
     * @var Post $post
     */
    $post = Post::find($id);
    $post->publish();
}
```

phpDoc comments can also help in complex cases:

```php
/**
 * @var Post[] $posts
 */
$posts = Post::all();
foreach($posts as $post) {
    $post->// Here, the IDE should provide
           // all methods and fields of the Post class
}
```

IDE hints are enjoyable while writing code, but more importantly, by providing these hints, the IDE understands their origin and will always find their usages.

If a function returns an object of a specific class, it should be declared as a return type (starting from PHP 7) or in the function's `@return` phpDoc tag.

```php
public function getPost($id): Post
{
    //...
}

/**
 * @return Post[] | Collection
 */
public function getPostsBySomeCriteria(...)
{
    return Post::where(...)->get();
}
```

I've been asked a couple of times: why am I turning PHP into Java? It's not exactly like that. I'm adding small comments to have convenient IDE hints right now and immense future assistance for navigation, refactoring, and debugging. Even for small projects, they are incredibly useful.

## Templates

Nowadays, more and more projects have only an API interface, but there is still a significant number of projects that directly generate HTML. They use templates that involve many method/field calls. A typical template call in Laravel looks like this:

```php
return view('posts.create', [
    'author' => \Auth::user(),
    'categories' => Category::all(),
]);
```

It looks like calling a virtual function. Compare it with this pseudocode:

```php
/**
 * @param User $author
 * @param Category[] | Collection $categories
 */
function showPostCreateView(User $author, $categories): string
{
    // template code
}

return showPostCreateView(\Auth::user(), Category::all());
```

It would be desirable to describe the template parameters like typical function parameters. It's easy when the templates are written in pure PHP – phpDoc comments would help. For template engines like Blade, it's not that straightforward and depends on the IDE. I work with PhpStorm, and it allows to declare types via phpDoc:

```php
<?php
/**
 * @var \App\Models\User $author
 * @var \App\Models\Category[] $categories
 */
?>

@foreach($categories as $category)
    {{$category->//Category class fields and methods autocomplete}}
@endforeach
```

I understand that many people might consider this excessive and a waste of time, but after all the efforts put into static "typing," my code becomes much more flexible. I can easily find all the usages of fields and methods and rename everything automatically. Each refactoring causes minimal pain.

## Model Fields
Using magic methods like **__get**, **__set**, **__call**, and others is tempting but dangerous. It won't be easy to find such magic calls. If you use them, it's better to provide these classes with the necessary phpDoc comments. Here's an example with a small Eloquent model:

```php
class User extends Model
{
    public function roles()
    {
        return $this->hasMany(Role::class);
    }
}
```

This class has several virtual fields representing the fields of the **users** table, as well as the **roles** field. With the laravel-ide-helper package, you can automatically generate phpDoc for this class. Just one artisan command call will generate comments for all models.

```php
/**
 * App\User
 *
 * @property int $id
 * @property string $name
 * @property string $email
 * @property-read Collection|\App\Role[] $roles
 * @method static Builder|\App\User whereEmail($value)
 * @method static Builder|\App\User whereId($value)
 * @method static Builder|\App\User whereName($value)
 * @mixin \Eloquent
 */
class User extends Model
{
    public function roles()
    {
        return $this->hasMany(Role::class);
    }
}

$user = new User();
$user->// all fields will be completed
```

Let's go back to the example from the previous chapter:

```php
public function store(Request $request, ImageUploader $imageUploader) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
    ]);
    
    $avatarFileName = ...;    
    $imageUploader->upload($avatarFileName, $request->file('avatar'));
        
    $user = new User($request->except('avatar'));
    $user->avatarUrl = $avatarFileName;
    
    if (!$user->save()) {
        return redirect()->back()->withMessage('...');
    }
    
    \Email::send($user->email, 'Hi email');
        
    return redirect()->route('users');
}
```
Creating a User entity looks a bit odd. Before some changes, it looked more elegant:

```php
User::create($request->all());
```

But then it had to be changed because the **avatarUrl** field cannot be directly assigned from the request object.

```php
$user = new User($request->except('avatar'));
$user->avatarUrl = $avatarFileName;
```

Not only does it look strange, but it's also unsafe. This method is used in a user registration process. A field **admin** may be added in the future, distinguishing administrators from regular users. A clever hacker could  add a new field to the registration form:

```html
<input type="hidden" name="admin" value="1"> 
```

And would become an administrator immediately after registration. For these reasons, some experts suggest listing all the required fields (there is also the **$request->validated()** method, but its shortcomings will be understood later in the book if you read it carefully):

```php
$request->only(['email', 'name']);
```

But if we list all the fields, why not make object creation more civilized?

```php
$user = new User();
$user->email = $request['email'];
$user->name = $request['name'];
$user->avatarUrl = $avatarFileName;
```

This code can already be shown in society. It will be understandable to any PHP developer. The IDE will always find where the **email** field of the **User** class was assigned a value.

"What if the entity has 50 fields?" It may be worth reconsidering the user interface a bit. Fifty fields are too much for anyone, whether a user or a developer. If you disagree, the book will show some techniques to reduce this code even for many fields.

## Laravel Idea

This was so important to me that I developed a plugin for PhpStorm called **Laravel Idea**. It understands Laravel magic well and eliminates the need for all the phpDoc comments I mentioned earlier. It offers a lot of code generation and hundreds of other features. Here are a few examples.

```php
User::where('email', $email);
```

The plugin virtually links the string `'email'` in this code to the field `$email` of the User class. It allows autocompletion of all entity fields for the first argument of the `where` method. It also finds all similar uses of the `$email` field and can automatically rename all such strings if it will be renamed to something like `$firstEmail`. This works even for complex cases:

```php
Post::with('author:email');

Post::with([
    'author' => function (Builder $query) {
        $query->where('email', 'some@email');
    }]);
```

PhpStorm will recognize that the `$email` field was used in both cases. The same goes for routing:

```php
Route::get('/', 'HomeController@index');
```

Here, there are references to the `HomeController` class and the `index` method. If you ask PhpStorm to find places where the `index` method is used, it will find the location in this route file. These seemingly unnecessary features allow for better control over the application, which is essential for medium or large-sized applications.

We have made our code more convenient for future refactoring or debugging. This "static typing" is not mandatory but extremely useful. It's worth giving it a try, at least.