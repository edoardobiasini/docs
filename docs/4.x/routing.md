# Routing

Routing helps Craft smartly handle requests to your site. When a request comes in to Craft, it checks to determine where to route the request.

The checks detailed below explain how this process works. This information can helpful to you when troubleshooting template loading, plugin action URLs, dynamic routes, and unexpected 404 errors.

Here is how Craft handles each request:


0. **Should Craft handle this request in the first place?**

   It’s important to keep in mind that Craft doesn’t get involved for *every* request that touches your server, but only requests that go to your `index.php` file.

   The `.htaccess` file that [comes with Craft](https://github.com/craftcms/craft/blob/master/web/.htaccess) will redirect all requests that don’t match a directory or file on your web server over to `index.php` behind the scenes. But if you point your browser directly at a file that *does* exist (such as an image, CSS, or JavaScript file), your web server will serve that file directly without loading Craft.

1. **Is it an action request?**

   Action requests either have a URL that begins with `actions/` (or whatever your <config4:actionTrigger> config setting is set to), or an `action` parameter in the POST request or the query string.

   Craft routes action requests to a controller action that perform actions. Craft has system Controller actions for core actions, but plugins may also have Controllers that define their own custom actions.

   The request doesn’t necessarily end after a controller call. The controller may allow it to keep going.

2. **Is it an element request?**

   If the URI matches an element’s URI, Craft lets the element decide how to route the request. For example, if an [entry’s](entries.md) URI is requested, then the entry will route the request to the template specified in its section’s settings, with an `entry` variable predefined, set to the requested entry.

   ::: tip
   Modules and plugins can override element routes using the [EVENT_SET_ROUTE](craft4:craft\base\Element::EVENT_SET_ROUTE) event.
   :::

3. **Does the URI match a route or URI rule?**

   If the URI matches any [dynamic routes](#dynamic-routes) or [URI rules](#advanced-routing-with-url-rules), the template or controller action specified by it will get loaded.

4. **Does the URI match a template?**

   Craft will check if the URI is a valid [template path](dev/README.md#template-paths). If it is, Craft will return the matched template.

   ::: tip
   If any of the URI segments begin with an underscore (e.g. `blog/_archive/index`), Craft will skip this step.
   :::

5. **404**

   If none of the above checks are successful, Craft will throw a [NotFoundHttpException](yii2:yii\web\NotFoundHttpException). \
   If [Dev Mode](config4:devMode) is enabled, an error report for the exception will be shown. Otherwise, a 404 error will be returned using either your [custom error template](#error-templates) or Craft’s own default.

## Dynamic Routes

In some cases you want a URL to load a template, but you don’t want the URI to match the template path.

A good example of this is a yearly archive page, where you want the year to be one of the segments in the URL (e.g. `blog/archive/2018`). It would be silly to create a new template for every year. Instead, you should set up a new _route_.

![Creating a New Route](./images/routing-creating-new-route.png)

### Creating Routes

To create a new Route, go to **Settings** → **Routes** and choose **New Route**. A modal window will appear where you can define the route settings:

- What should the URI look like?
- Which template should get loaded?

The first setting can contain “tokens”, which represent a range of possible matches, rather than a specific string. (The `year` token, for example, represents four consecutive digits.) When you click on a token, Craft inserts it into the URI setting wherever the cursor is.

If you want to match URIs that look like `blog/archive/2018`, type `blog/archive/` into the URI field and choose the `year` token.

::: tip
Route URIs should _not_ begin with a slash (`/`).
:::

After defining your URI pattern and entering a template path, press **Save**. The modal will close, revealing your new route on the page.

When you point your browser to `https://my-project.tld/blog/archive/2018`, it will match your new route, and Craft will load the specified template.

The value of the `year` token will also be available to the template as a variable called `year`.


### Available Tokens

The following tokens are available to the URI setting:

- `*` – Any string of characters, except for a forward slash (`/`)
- `day` – Day of a month (`1`-`31` or `01`-`31`)
- `month` – Numeric representation of a month (1-12 or 01-12)
- `number` – Any positive integer
- `page` – Any positive integer
- `uid` – A v4 compatible UUID (universally unique ID)
- `slug` – Any string of characters, except for a forward slash (`/`)
- `tag` – Any string of characters, except for a forward slash (`/`)
- `year` – Four consecutive digits


## Advanced Routing with URL Rules

In addition to routes, you can define [URL rules](https://www.yiiframework.com/doc/guide/2.0/en/runtime-routing#url-rules) in `config/routes.php`.

```php
return [
    // Route blog/archive/YYYY to a controller action
    'blog/archive/<year:\d{4}>' => 'controller/action/path',

    // Route blog/archive/YYYY to a template
    'blog/archive/<year:\d{4}>' => ['template' => 'blog/_archive'],
];
```

If your Craft installation has multiple sites, you can create site-specific URL rules by placing them in a sub-array, and set the key to the site’s handle.

```php
return [
    'siteHandle' => [
        'blog/archive/<year:\d{4}>' => 'controller/action/path',
    ],
];
```

Craft also supports special tokens that you can use within the regular expression portion of your [named parameters](https://www.yiiframework.com/doc/guide/2.0/en/runtime-routing#named-parameters):

- `{handle}` – matches a field handle, volume handle, etc.
- `{slug}` – matches an entry slug, category slug, etc.
- `{uid}` – matches a v4 UUID.

```php
return [
    'blog/<entrySlug:{slug}>' => 'controller/action/path',
];
```

### Accessing Named Parameters in your Templates

URL rules that route to a template (`['template' => '<TemplatePath>']`) will pass any matched named parameters to the template as variables.

For example, with this URL rule:

```php
'blog/archive/<year:\d{4}>' => ['template' => 'blog/_archive'],
```

If you access `https://my-project.tld/blog/archive/2018`, your `blog/_archive.twig` template will get loaded a `year` variable set to `2018`.

```twig
<h1>Blog Entries from {{ year }}</h1>
```

::: tip
If you define a wildcard token (`*`) in the control panel, it will automatically be available as a named parameter called `any`.

![Screenshot of Create a new route control panel form with wildcard token](./images/route-with-wildcard-token.png =400x300)

The template for `my-project.tld/foo/some-slug` could then use `{{ any }}`:

```twig
It seems you’re looking for `{{ any }}`.
{# output: It seems you’re looking for `some-slug`. #}
```
:::

### Accessing Named Parameters in your Controllers

To access named parameters in your controllers, you will need to add the parameter(s) to your controller’s action definition.

For example, with this URL rule:

```php
'blog/archive/<year:\d{4}>' => 'controller/action/foo',
```

Your controller method would be:

```php
public function actionFoo(int $year = null)
{
    // ...
}
```

## Error Templates

You can provide your own error templates for Craft to use when returning errors on the front end.

When an error is encountered, Craft will look for a template in your `templates/` directory, in the following order:

1. A template matching the error’s status code, like `404.twig`.
2. For a 503 error, a template named `offline.twig`.
3. A template named `error.twig`.

::: tip
You can tell Craft to look for the error template in a nested template directory, using the <config4:errorTemplatePrefix> config setting.
:::

If Craft finds a matching error template, it will render it with the following variables:

- `message` – error message
- `code` – exception code
- `file` – file that threw the exception
- `line` – line in which the exception occurred
- `statusCode` – error’s HTTP status code

::: tip
Custom error templates are only used when [Dev Mode](config4:devMode) is **disabled**. When it’s enabled, an exception view will be rendered instead.
:::
