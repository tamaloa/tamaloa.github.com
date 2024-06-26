---
layout: post
title: "Diffing a fresh rails 3 app against a rails 4 app"
excerpt: "A visual diff comparing the application directory structure and files created by 'rails new' for rails 3.2.14 and rails 4.0.0"
description: ""
category: 
tags: [rails, rails4]
---
{% include JB/setup %}

In the process of migrating to rails 4 i like to know what has changed
for the "default" app. The easiest way to check is to generate two new
rails apps using the different rails versions and then comparing with a
diff tool (i like meld :).

    gem install rails -v 3.2.14
    rails new rails-3.2.14
    gem install rails -v 4.0.0
    rails new rails-4.0.0

    meld rails-3.2.14 rails-4.0.0

The screenshot shows not too many large differences in the directory
structure or filewise. Noticable is

-   .gitkeep was renamed to .keep which i think is nice because we use
    mercurial :)
-   a default index page (with corresponding image) no longer exists ..
    phew, saves deleting it every time
-   the new concerns dir is added to app/controllers and app/models
-   executables now have their place in bin
-   config for filter_paramter_logging was added
-   there no longer is a doc dir (who used it anyway?)
-   test dirs now correspond to what is tested (unit  model, functional 
    controller) and new ones for helper and mailer testing were added.
-   and of course plugins dir has finally been removed (remember all
    those deprecation warnings :)

![A diff of a newly created rails 3 vs rails 4 app]({{ site.url }}/assets/images/rails-3.2.14_vs_rails-4.0.0_diff.png)


### Conclusion

...some time later :)
