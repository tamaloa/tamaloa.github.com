# Jekyll-Bootstrap

The quickest way to start and publish your Jekyll powered blog. 100% compatible with GitHub pages

## Usage

### Run Jekyll Locally

    $ jekyll serve

Your blog is now available at: http://localhost:4000/.

### Create a Post

Create posts easily via rake task:

    $ rake post title="Hello World"

The rake task automatically creates a file with properly formatted filename and YAML Front Matter. Make sure to specify your own title. By default, the date is the current date.

The rake task will never overwrite existing posts unless you tell it to.


### Publish

After you've added posts or made changes to your theme or other files, simply commit them to your git repo and push the commits up to GitHub.
Then build the changes locally and push to gh-pages branch using the jgd gem

    $ jgd
