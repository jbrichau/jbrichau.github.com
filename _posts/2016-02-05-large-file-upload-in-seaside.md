---
layout: post
title: Large file uploads in Seaside
category: Coding
tags:
  - Seaside
  - Smalltalk
year: 2016
month: 2
day: 5
published: true
summary: Using the Nginx file upload module to support (large) file uploads in Seaside web applications
image: post_four.jpg
---

Uploading files in a Seaside web application is [easy](http://book.seaside.st/book/fundamentals/forms/fileupload). Unfortunately, there is a drawback to the easiness: entire files are loaded into the Seaside backend's memory because an instance of `WAFile` contains the entire file's contents. In many situations, loading the file contents in the Seaside backend's memory is not necessary (for example, if the file only needs to be stored on disk) or even impossible (e.g. in the case of extremely large files).

This post details an extension to Seaside that works together with the [NGINX](http://www.nginx.org) front-end web server and its [file upload module](https://www.nginx.com/resources/wiki/modules/upload/). The uploaded file is stored to disk and NGINX subsequently only passes a file reference to the Seaside backend. This off-loads the heavy lifting to the web server and prevents memory overload in the Seaside backend while still keeping close to the ease of implementation of "traditional" file uploads in Seaside.

Many of you will notice that this solution is based on work by Nick Ager, whose blog post has unfortunately disappeared from the web. Since there have been several questions on the [Seaside mailinglist](http://lists.squeakfoundation.org/cgi-bin/mailman/listinfo/seaside) on this topic, I thought it would a good idea to revisit our implementation (which has been working in production for years now) and make it usable as a separate Seaside extension.

## Seaside-ExternalFileUpload
The [Seaside](http://www.seaside.st/) extension to support large file uploads is in the optional [Seaside-ExternalFileUpload package](http://www.smalltalkhub.com/#!/~Seaside/Seaside32/packages/Seaside-ExternalFileUpload). At the time of writing of this post, you need to load this package manually in your Seaside3.x image. The package should work well in 3.0, 3.1 and 3.2.

## NGINX with file upload module
You need to compile NGINX from source because, like many of its modules, the file upload module is not included in the binary distributions. Also, since NGINX 1.3.0 or so, you need to make sure to use version 2.2 of the file upload module. Executing the following commands should work for you, but mind you might need to pass additional configuration options to the `configure` command to fit your NGINX setup.

{% highlight bash %}
  sudo curl -O http://nginx.org/download/nginx-1.8.1.tar.gz
  sudo tar xf nginx-1.8.1.tar.gz
  cd nginx-1.8.1
  sudo curl -L -o nginx-upload-module-2.2.0.tar.gz https://github.com/vkholodkov/nginx-upload-module/tarball/2.2
  sudo tar xf nginx-upload-module-2.2.0.tar.gz
  sudo mv vkholodkov-nginx-upload-module-aba1e3f nginx-upload-module-2.2.0
  sudo ./configure --add-module=./nginx-upload-module-2.2.0/
  sudo make install
{% endhighlight %}

## NGINX configuration
Once you get NGINX installed, you need to configure an upload location in the server block that concerns your Seaside app. The following configuration defines that location as the path `/fileupload`, which means that the file upload plugin is listening at that location. Files uploaded to that location will be stored in the `upload_store` directory on the server. In our case, the files will be uploaded to `/var/www/uploadstore`.

Once the file is uploaded, the request that is sent to the Seaside back end (listening at location `/`) has the additional fields `name`, `content_type` and `path` that contain the respective properties of the uploaded file. These properties will be available in the Seaside callback attached to the file upload field. Finally, the configuration also ensures all (other) fields of the form are sent to Seaside such that all callbacks of the form are executed there.

Please see the [file upload module documentation](https://www.nginx.com/resources/wiki/modules/upload/) for more information on these and other configuration parameters.

{% highlight nginx %}
# Upload form should be submitted to this location
location ~ /fileupload {

  # Pass altered request body to this location
  upload_pass /;

  error_page 405 415 = /;

  # Store files to this directory
  upload_store /var/www/uploadstore;

  # Allow uploaded files to be read only by user
  upload_store_access user:rw group:rw all:rw;

  # Set specified fields in request body
  upload_set_form_field $upload_field_name "";
  upload_set_form_field $upload_field_name.name "$upload_file_name";
  upload_set_form_field $upload_field_name.content_type "$upload_content_type";
  upload_set_form_field $upload_field_name.path "$upload_tmp_path";

  # seaside automatically assigns sequential integers to fields with callbacks
  # we want to pass those fields to the backend
  upload_pass_form_field "^\d+$";

  upload_cleanup 400 404 499 500-505;
}
{% endhighlight %}

## Example File Upload
The package `Seaside-ExternalFileUpload` contains an example component `WAFileUploadExample` that demonstrates "traditional" file uploads side-by-side with the new "external" file uploads. Here is the snippet for such an external file upload (i.e. where the upload is handled by the front-end web server NGINX):

{% highlight smalltalk %}
html form
  multipart;
  fileUploadLocation: 'fileupload';
  with: [
      html externalFileUpload
          callback: [ :ef | file := ef ].
      html submitButton
          text: 'Upload file via front-end'
  ].
{% endhighlight %}

Like any form with a file upload field, you need to set it to be `multipart`. Next, you need to pass the `fileUploadLocation`, which is the location configured in NGINX to handle file uploads. In our case, this is `fileupload`, but you can choose any name you want for that location as long as you use the same name here and in the NGINX configuration.
The file upload field tag is `externalFileUpload` (instead of `fileUpload`). The callback block of this field `[ :ef | file := ef ]` will be invoked with a `WAExternalFile` instance instead of a `WAFile` instance. A `WAExternalFile` contains the uploaded file's filename, its content type and path on disk. From here on, it's up to you what to do with the file. A good idea is to move the file to its proper location, for example.

The `Seaside-ExternalFileUpload` package is currently a preview package. It will evolve as we integrate it further, for example by adding support for the ajax file uploads (as they are part of Seaside 3.2) and the [jQuery file upload plugin](https://blueimp.github.io/jQuery-File-Upload/). More about this in upcoming posts.

Please contact us on the [Seaside mailinglist](http://lists.squeakfoundation.org/cgi-bin/mailman/listinfo/seaside) in case you need help or for any additional comments and remarks.
