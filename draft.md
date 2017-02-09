
#User Authorization in Laravel 5.3 with Spatie Laravel-Permission

This tutorial would show how to set up a  Role-based Permissions system using this [Spatie Laravel-Permission](https://github.com/spatie/laravel-permission). A very simple blog application would be created to see it in action.

##Development Environment and Installation
You may set up your development environment using [Homestead](https://www.sitepoint.com/quick-tip-get-homestead-vagrant-vm-running/).  After installing Laravel-Permission you would be able to add permissions to a user, assign permissions to roles and much more. To install the package run

```bash
composer require spatie/laravel-permission
```
   
Next include the package to our list of service providers, in `config/app.php` add `Spatie\Permission\PermissionServiceProvider::class` so our file would look like this

```
'providers' => [
	...
	Spatie\Permission\PermissionServiceProvider::class,
];
```
Next  publish the migration file with the command

```bash
php artisan vendor:publish --provider=`Spatie\Permission\PermissionServiceProvider` --tag=`migrations`
```

The migration schema contains relationship information between a "user_has_permissions table" and the default "users" table. If you don't want your users table to be named "users", edit the migration file you just published. Next create the database and update the `.env` file to include the database information. To build the tables, run
    
```bash
php artisan migrate
```
    
Now if you look into the database you would have the following tables:

 - `migrations`: This keeps track of migration process that have ran
 - `users`: This holds the users data of the application
 - `password_resets`: Holds token information when users request a new password
 - `permissions`: This holds the various permissions needed in the application
 - `roles`: This holds the roles in our application
 - `role_has_permission`: This is a pivot table that holds relationship information between the permissions table and the role table
 - `user_has_roles`: Also a pivot table, holds relationship information between the roles and the users table.
 -  `user_has_permissions`: Also a pivot table, holds relationship information between the users table and the permissions table.

Publish the configuration file for this package by running

```bash
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider" --tag="config"
```
The config file lets us to set the location of the eloquent model of the permission and role class, you can also manually set the table names that should be used to retrieve your roles and permissions.

Next you should go ahead and install [Laravel HTML collective](https://laravelcollective.com/docs/5.3/html) as this would be useful further on


##Usage
To allow us to assign permissions to users and roles add the 
`HasRoles` trait to the User model:

```php
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable {
	use HasRoles;

	// ...
}
```

That's all the installation and configuration needed. A role can be created like a regular Eloquent model, like this:

```php
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

$role = Role::create(['name' => 'writer']);
$permission = Permission::create(['name' => 'edit articles']);
```

Now you can get the permissions associated to a user like this:

```php
$permissions = $user->permissions;
```

And using the [pluck method `pluck()`](https://laravel.com/docs/5.2/collections#method-pluck) you can get the role names associated with a user, like this

```php
$roles = $user->roles()->pluck('name');
```

Other methods available to us include:

 - `givePermissionTo()`: Allows us to give persmission to a user or role
 - `revokePermissionTo()`: Revoke permission from a user or role
 - `hasPermissionTo()`: Check if a user or role has a given permission
 - `assignRole()`: Assigns role to a user
 - `removeRole()`: Removes role from a user
 - `hasRole()`: Checks if a user has a role
 - `hasAnyRole(Role::all())`: Checks if a user has any of a given list of roles
 - `hasAllRoles(Role::all())`: Checks if a user has all of a given list of role
 
The methods `assignRole`, `hasRole`, `hasAnyRole`, `hasAllRoles` and `removeRole` can accept a string, a `Spatie\Permission\Models\Role-object` or an `\Illuminate\Support\Collection` object. The `givePermissionTo` and `revokePermissionTo` methods can accept a string or a `Spatie\Permission\Models\Permission` object.

##Blade Directives

Laravel-Permission also allows to use blade directives to verify if the logged in user has all or any of a given list of roles:

```
@role('writer')
	I'm a writer!
@else
	I'm not a writer...
@endrole

@hasrole('writer')
	I'm a writer!
@else
	I'm not a writer...
@endhasrole

@hasanyrole(Role::all())
	I have one or more of these roles!
@else
	I have none of these roles...
@endhasanyrole

@hasallroles(Role::all())
	I have all of these roles!
@else
	I don't have all of these roles...
@endhasallroles
```

The blade directives above depends on the users role. However in our application, you need to check directly in our view if a user has a certain permission, that is you want to be able to do something like:

```
@haspermission('Edit Post')
	I have permission to edit
@endhaspermission
```

To create this custom conditional, go to `\vendor\spatie\laravel-permission\src\PermissionServiceProvider.php` and add this code:

```php
protected function registerBladeExtensions() {
	...
	$bladeCompiler->directive('haspermission', function($permissions) {
		return "<?php if(auth()->check() && auth()->user()->hasPermissionTo({$permissions})): ?>";
	});
	$bladeCompiler->directive('endhaspermission', function () {
		return '<?php endif; ?>';
	}); 
}
```

Now you can use the blade directive `@haspermission()` any where in our view.

##Simple Blog Application

Let's build a simple blog to demonstrate how Laravel-Permission could be used in an application. In addition to allowing users to view posts, our application would have a role-based permission system and only roles with the right permission would be allowed to edit or delete posts. You would need a total of four controllers for this application. Let's use resource controllers, as this automatically adds stub methods for us. Our controllers would be called

 1. PostController
 2. UserController
 3. RoleController
 4. PermissionController
 
Before working on these controllers let's create our authentication system. With one command Laravel provides a quick way to scaffold all of the routes and views needed for authentication.

```bash
php artisan make:auth
```
   
This command also creates a `HomeController` (you can delete this as it won't be needed),  a `resources/views/layouts/app.blade.php` file which contains markup that would be shared by all our views and a `app/Http/Controllers/Auth` directory which contains the controllers for registration and login. Switch into this directory and open the `RegisterController.php`file. Replace the `create` method with

```php
protected function create(array $data)
{
    return User::create([
        'name' => $data['name'],
        'email' => $data['email'],
        'password' => $data['password'],
    ]);
}  
```

Here the `bcrypt` function, used for hashing our password was removed. Instead let's define a mutator in `App\User.php` which would encrypt all our password fields. In `App\User.php` add this method

```php
public function setPasswordAttribute($password)
{   
    $this->attributes['password'] = bcrypt($password);
}
```

This would provide the same functionality as before but now you don't need to write the `bcrypt` function when dealing with the password field in subsequent controllers.

Also in the `RegisterController.php`file. Change the `$redirectTo` property to

```php
protected $redirectTo = '/';
```

Do the same thing in the `LoginController.php`file. 

Since the `HomeController` has been deleted our users are now redirected to the home page which would contain a list of our blog posts.
 
Next let's edit the `resources/views/layouts/app.blade.php` file changing the title of all pages to 'ACL Demo', also include an extra drop-down Admin link to view all users, and a custom `styles.css` which would have extra styling for our `resources/views/posts/index.blade.php` view. The Admin link would only be viewed by Users with the 'Admin' Role, the CSS just adds a paragraph in the teaser of our index view, the file should be located in `public/css/styles.css` The `resources/views/layouts/app.blade.php` file should look like this:
   
```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  
  <title>ACL Demo</title>
  
  <!-- Fonts -->
  <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.4.0/css/font-awesome.min.css" rel='stylesheet' type='text/css'>
  <link href="https://fonts.googleapis.com/css?family=Lato:100,300,400,700" rel='stylesheet' type='text/css'>
  
  <!-- Styles -->
  <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet">
  {!! Html::style('css/styles.css') !!}
  {{-- <link href="{{ elixir('css/app.css') }}" rel="stylesheet"> --}}
  
  <style>
    body {
      font-family: 'Lato';
    }
    
    .fa-btn {
      margin-right: 6px;
    }
  </style>
</head>
<body id="app-layout">
  <nav class="navbar navbar-default navbar-static-top">
    <div class="container">
      <div class="navbar-header">
        
        <!-- Collapsed Hamburger -->
        <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#app-navbar-collapse">
          <span class="sr-only">Toggle Navigation</span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
        </button>
        
        <!-- Branding Image -->
        <a class="navbar-brand" href="{{ url('/') }}">
          ACL Demo
        </a>
      </div>
      <div class="collapse navbar-collapse" id="app-navbar-collapse">
        <!-- Left Side Of Navbar -->
        <ul class="nav navbar-nav">
          <li><a href="{{ url('/') }}">Home</a></li>
          @if (!Auth::guest())
          <li><a href="{{ route('posts.create') }}">New Article</a></li>
          @endif
        </ul>
        
        <!-- Right Side Of Navbar -->
        <ul class="nav navbar-nav navbar-right">
          <!-- Authentication Links -->
          
          @if (Auth::guest())
          <li><a href="{{ url('/login') }}">Login</a></li>
          <li><a href="{{ url('/register') }}">Register</a></li>
          @else
          <li class="dropdown">
            <a href="#" class="dropdown-toggle" data-toggle="dropdown" role="button" aria-expanded="false">
              {{ Auth::user()->name }} <span class="caret"></span>
            </a>
            <ul class="dropdown-menu" role="menu">
              <li>@role('Admin')
                <a href="{{ url('/users') }}"><i class="fa fa-btn fa-unlock"></i>Admin</a>
                @endrole
                <a href="{{ url('/logout') }}"><i class="fa fa-btn fa-sign-out"></i>Logout</a></li>
              </ul>
            </li>
            @endif
          </ul>
        </div>
      </div>
    </nav>
    @if(Session::has('flash_message'))
    <div class="container">      
      <div class="alert alert-success"><em> {!! session('flash_message') !!}</em></div>
    </div>
    @endif    
    @yield('content')
        
    <!-- JavaScripts -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js"></script>
    {{-- <script src="{{ elixir('js/app.js') }}"></script> --}}
  </body>
</html>
```
	
And the `styles.css` file is simply:

```css
p.teaser {
	text-indent: 30px; 
	}
```

##Post Controller

First, let's create the migration and model files for the `PostController`

```bash
php artisan make:model Post -m
```
This command generates a migration file in `app/database/migrations` for generating a new MySQL table named posts in our database, and a model file `Post.php`in the `app` directory. Let's edit the migration file to include title and body fields of our post.  After editing the migration file, it should look like this: 
   
```php
<?php
//database\migrations\xxxx_xx_xx_xxxxxx_create_posts_table.php

use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreatePostsTable extends Migration {
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up() {
        Schema::create('posts', function (Blueprint $table) {
            $table->increments('id');
            $table->string('title');
            $table->text('body');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down() {
        Schema::drop('posts');
    }
}
```

After saving the file, run

```bash
php artisan migrate
```

You can now check the database for the post table and columns. 

![enter image description here](https://goo.gl/pRvNg1 "acl_db.jpg")

Next edit the `Post.php` file to look like this

```php
namespace App;
use Illuminate\Database\Eloquent\Model;

class Post extends Model {
	protected $fillable = [
		'title', 'body'
	];
}
```
Now let's generate our resource controller.
  
```bash
php artisan make:controller PostController --resource
```
    
This will create our controller with all the stub methods needed. Edit this file to look like this

```php
<?php
// app/Http/Controllers/PostController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests;
use App\Post;
use Auth;
use Session;

class PostController extends Controller {

    public function __construct() {
        $this->middleware(['auth', 'clearance'])->except('index', 'show');
    }

    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index() {
        $posts = Post::orderby('id', 'desc')->paginate(5);

        return view('posts.index', compact('posts'));
    }

    /**
     * Show the form for creating a new resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function create() {
        return view('posts.create');
    }

    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request) {    
        $this->validate($request, [
            'title'=>'required|max:100',
            'body' =>'required',
            ]);

        $title = $request['title'];
        $body = $request['body'];

        $post = Post::create($request->only('title', 'body'));

        return redirect()->route('posts.index')
            ->with('flash_message', 'Article,
             '. $post->title.' created');
    }

    /**
     * Display the specified resource.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show($id) {
        $post = Post::findOrFail($id);

        return view ('posts.show', compact('post'));
    }

    /**
     * Show the form for editing the specified resource.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function edit($id) {
        $post = Post::findOrFail($id);

        return view('posts.edit', compact('post'));
    }

    /**
     * Update the specified resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, $id) {
        $this->validate($request, [
            'title'=>'required|max:100',
            'body'=>'required',
        ]);

		$post = Post::findOrFail($id);
        $post->title = $request->input('title');
        $post->body = $request->input('body');
        $post->save();

        return redirect()->route('posts.show', 
            $post->id)->with('flash_message', 
            'Article, '. $post->title.' updated');
        
    }

    /**
     * Remove the specified resource from storage.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function destroy($id) {
        $post = Post::findOrFail($id);
        $post->delete();

        return redirect()->route('posts.index')
            ->with('flash_message',
             'Article successfully deleted');

    }
}
```

Here the `Post` class was imported from our model and the `Auth` class which was generated with the `make:auth` command earlier. These were imported so that you would be able to make eloquent queries on the `Post` table and so as to be able to have access to authentication information of our users. In the constructor two middlewares were called, one is `auth` which restricts access to the `PostController` methods to authenticated users the other is a custom middleware is yet to be created. This would be responsible for our Permissions and Roles system.  Next, index and show are passed into the except method to allow all users to be able to view posts.

The `index()` method lists all the available Posts. It queries the `Post` table for all posts and passes this information to the view. `Paginate()` allows us to limit the number of posts in a page, in this case five.  

The `create()` method simply returns the `posts/create` view which would contain a form for creating new posts. The `store()` method saves the information input from the `posts/create` view. The information is first validated and after it is saved, a flash message is passed to the view `posts/index`.

Our `show()` method of the `PostController` allows us to display a single post. This method takes the post id as an argument and passes it to the method `Post::find()`. The result of the query is then sent to our `posts/show` view.

The `edit()` method, similar to the `create()` method simply returns the `posts/edit` view which would contain a form for creating editing posts. The `update()` method takes the information from the  `posts/edit` view and updates the record. The `destroy()` method let's us delete a post. 

Now that you have the `PostController` you need to set up the routes. Edit your `app/routes/web.php` file to look like this

```php
<?php

Route::get('/', [
    'as' => 'home', 'uses' => 'PostController@index'
]);

Auth::routes();

Route::get('logout', '\App\Http\Controllers\Auth\LoginController@logout');

Route::resource('users', 'UserController');

Route::resource('roles', 'RoleController');

Route::resource('permissions', 'PermissionController');

Route::resource('posts', 'PostController');
```

The `/` route  is the route to our home page, here it was renamed to `home` The `Auth` route was generated when you ran the `make:auth` command. It handles authentication related routes. The `logout`route calls the `logout`method and it allows us to be able to logout from our application. The other three routes are for resources that would be created later.

