Title: How we use Github as our CMS
Subheading: Using Github as a CMS, we publish pages to our Laravel driven site automatically.
Published: Feb. 25th, 2021
Tags: laravel, github, cms, markdown
Author: Dan Cameron
Featured Image: /2021/02/PageController.php-thumb.webp
Excerpt: Using Github as a CMS to edit, provide comments, and automatically deploy new articles/posts with multiple authors.

### Why we Decided to Use Github as a CMS

Did you know that Laravel manages their [documentation](https://laravel.com/docs/8.x/installation) via a [Github repo](https://github.com/laravel/docs)? It's pretty cool that they're able to receive pull requests to handle update requests *from anyone*, allowing for easy collaboration. While I was exploring this idea for Apazed documentation I found out they not only use Github to collaborate on the documentation, but they use Github as a CMS, serving the files directly (i.e. no copy/pasting into some other system). 

If you head over the the [laravel.com-next](https://github.com/laravel/laravel.com-next) repo you'll see exactly how they do it. Including the awesome way they use tags for documentation versions based on major Laravel releases, and search. 

I soon figured I wanted to use Github for more than documentation, including articles (like this one) and content pages (i.e. [About Us](/about-us)). Content that I contemplated using WordPress for, a system without the ability to allow for collaboration the way Github provides. I'm hoping posts like this, especially those that are developer focused will greatly benefit from the ability for a pull request. I make *a lot of mistakes* and submitting a pull request seems fun, no?

### How Apazed Implemented Github as a CMS for Laravel

I'll be referencing both the github repo for [laravel.com-next](https://github.com/laravel/laravel.com-next) and sharing some important code snippets that should help you get this done on your own. 

**1. Create a Separate Git repo**

This is the easy part, for you and especially for me to write. If you're overwhelmed with this or not sure what I mean [here's an article you you'll enjoy more](/articles/culture/oswald-the-octopus).

**2. Add the Repo as a Submodule**

If you're not familiar with git submodules [here's a great post](https://github.blog/2016-02-01-working-with-submodules/) to get you started. And, I'll go over how to automate page updates via commit hooks in a bit. 

**3. Create the Pages Controller (and routes)**

[Here's the controller](https://github.com/laravel/laravel.com-next/blob/master/app/Http/Controllers/DocsController.php) that Laravel uses. Nothing special. For us we use Inertia, so here's our ```show()``` for our pages.

```php
public function showPage($page = null)
{
    $page = $this->pages->get( $page, 'pages' );

    if (is_null( $page )) {
        // handle 404, or redirect.
    }

    return Inertia::render( 'StaticContent/Page', [
        'title' => $page['title'],
        'subtitle' => $page['subtitle'],
        'content' => $page['content'],
        'urls' => $page['urls']
    ] );
}
```

Obviously create your routes; for our articles I'm passing the category key and slug, the category is useful if you want to create [category pages](/articles/dev-diary).  

**4. Create a Pages Model**

Before you ```artisan make:model``` you'll want to create a new class, since we're going to use the Filesystem. Here's an [example to copy](https://github.com/laravel/laravel.com-next/blob/master/app/Documentation.php) and the big changes I made were parsing the file (similar to how they replaceLinks.

```php
if ($this->files->exists( $bp )) {
    $p = $this->parseFile( $this->files->get( $bp ) );
```

I wanted the markdown files to include the title, subtitle, and much more for articles. For example, this is [the markdown file for this article](https://github.com/apazed-com/pages/blob/master/articles/dev-diary/laravel-plus-github-content-publishing.md)

Here's an example of how I'm parsing the file for the Title as an example. 

```php
// Tightel: A Title
if (preg_match( '|Tightel:(.*)|i', $fileText, $_title )) {
    $title = strip_tags( $_title[1] );
    $fileText = preg_replace( '|Tightel:(.*)|i', '', $fileText );
} else {
    $title = NULL;
}
```

<small>I misspelled Title intentionally, so the filter wouldn't mess with it.</small>

After a lot of extracting/parsing of the file, an array is returned with the content and its information.

```php
return [
    'title' => trim( $title ),
    'subtitle' => trim( $subtitle ),
    'author' => trim( $author ),
    'excerpt' => trim( $excerpt ),
    'dates' => $dates,
    'tags' => $tags,
    'tweetId' => trim( $tweet ),
    'featuredImage' => '/_pages/images' . trim( $featuredImage ),
    'content' => (new Parsedown())->text( $fileText )
    // ...
];
```

**5. Markdown Parsing**

Notice that using Parsdown to filter the Markdown. Laravel is doing [the same to handle urls](https://github.com/laravel/laravel.com-next/blob/master/app/Parsedown.php). We're using a new Parsedown to add classes to images, set image paths correct, and manipulate link styling.  

```php
$text = str_replace( 'imgsrctag', 'img class="my-12 w-full ..." src="/_pages/images', $text );
$text = str_replace( 'atag', '<a class="text-link hover:text-linkHover"', $text );
// ...
```

**6. Wrap Up**

That's really it, the files on the system will parsed as markdown, additional information can be added to the file to help with metadata like titles, the view will then use that information to serve. At this point it's up to you to create the templates/components to display the content.

### You Promised Automation

We're using [Laravel Forge](https://forge.laravel.com/) for server management, and it allows for automatic deployments. I wanted the same for commits to the pages directory for Apazed, so I updated the deployment script to pull submodules and used the deployment trigger url for a Github webhook.

**Updated deployment script:**

```shell
...
git pull origin master
git submodule update --remote
...
```

**Updated Github webhooks:**

![Github Webhooks](/2021/02/github-trigger-1200.webp)

Now commits to the repo will trigger a deployment on our production server and updates are live within minutes.


### Added Tips

**Tailwind and PurgeCSS**

If you use Tailwind CSS, if not you should, you'll may want to have PurgeCSS scan the content directory before unused classes are treeshaken.

```js
purge: [
    './storage/framework/views/*.php',
    './resources/views/**/*.blade.php',
    './resources/js/**/*.vue',
    './public/_pages/**/*.md',
    './vendor/laravel/spark-stripe/resources/js/**/*.vue'
],
```

**Images**

We have an [images directory](https://github.com/apazed-com/pages/tree/master/images) that stores all of the images, and Parsedown fixes the paths to the images so that when writing I can use a simple path, i.e. ```/2021/02/github-trigger-1200.webp```.


