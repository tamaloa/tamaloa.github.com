# Jekyll-Bootstrap

The quickest way to start and publish your Jekyll powered blog. 100% compatible with GitHub pages

## Usage

### Run Jekyll Locally


### Create a Post

    $ jekyll serve

Your blog is now available at: http://localhost:4000/.

### Create a Post

Create posts easily via rake task:

    $ rake post title="Hello World"

The rake task automatically creates a file with properly formatted filename and YAML Front Matter. Make sure to specify your own title. By default, the date is the current date.

The rake task will never overwrite existing posts unless you tell it to.


### Publish

After you've added posts or made changes to your theme or other files, simply commit them to your git repo and push the commits up to GitHub.

    $ git add .
    $ git commit -m "Add new content"
    $ git push origin master

A GitHub post-commit hook will automatically deploy your changes to your hosted blog. You will receive a success or failure notice for every commit you make to your blog.


## Version

0.3.0 - stable and versioned using [semantic versioning](http://semver.org/).


## License

[MIT](http://opensource.org/licenses/MIT)
