
This is a simple patch to add `PUT` support for original [nginx-upload-module](http://www.grid.net.ru/nginx/upload.en.html)

## Compiling and Installing


### Nginx
	
	curl -LO http://nginx.org/download/nginx-1.3.4.tar.gz
	tar xf nginx-1.3.4.tar.gz 

### Upload module (with PUT support)

	git clone git://github.com/shairontoledo/ngx-upload-module.git


### Compiling  

  cd nginx-1.3.4

> for Mac, install pcre `brew install pcre` and add to `./configure` `--with-pcre --with-cc-opt=-I/usr/local/include --with-ld-opt=-L/usr/local/lib
`

    export NGX_PREFIX=/path/to/install

    ./configure --prefix=$NGX_PREFIX \
      --add-module=../nginx-upload-module  \
      --with-debug \
      --with-http_stub_status_module \
      --with-http_flv_module \
      --with-http_ssl_module \
      --with-http_dav_module \
      --with-http_gzip_static_module \
      --with-http_realip_module \
      --with-mail \
      --with-mail_ssl_module \
      --with-ipv6 

## Prepare directories 


     for ((i=0;i<10;i++)); do sudo mkdir -p /tmp/uploads/$i; done
     chmod 777 -R /tmp/uploads # set it properly to your system


## Sample nginx.conf with Unicorn for Rails

    worker_processes  1;

    error_log  logs/error.log  debug;

    events {
        worker_connections  1024;
    }

    http {

        upstream upstream_upload {
            server localhost:3000;
        }

        include       mime.types;
        default_type  application/octet-stream;

        sendfile        on;

        keepalive_timeout  65;

        #gzip  on;

        server {
            listen       80;
            server_name  localhost;

            location / {
                root   html;
                index  index.html index.htm;
            }

            location @unicorn_for_upload {
                proxy_pass http://upstream_upload;
            }

            location /files {
               
                #tryfiles 
                if (-f $request_filename) {
                    break;
                }

                if (-f $request_filename/index.html) {
                    rewrite (.*) $1/index.html break;
                }

                if (-f $request_filename.html) {
                    rewrite (.*) $1.html break;
                }
                
                # client size
                client_max_body_size 5632m;
                client_body_timeout 120;
                client_body_buffer_size 1m;
                send_timeout 120;

                #end point settings
                proxy_read_timeout 120;
                proxy_max_temp_file_size 6144m;
                proxy_send_timeout 120;

                #If it not a file, pass it to webapp
                if ($http_content_type = 'application/x-www-form-urlencoded'){
                   proxy_pass http://upstream_upload;
                  break;
                }

                if ($request_method ~* "POST|PUT") {
                    upload_store_access user:rw group:rw all:rw;
                    upload_store /tmp/uploads 1;
                    upload_max_file_size 6G;
                    upload_cleanup 400 404 499 500-505;

                    upload_aggregate_form_field "file_entry[filename]" "$upload_file_name";
                    upload_aggregate_form_field "file_entry[size]" "$upload_file_size";
                    upload_aggregate_form_field "file_entry[file_path]" "$upload_tmp_path";
                    upload_aggregate_form_field "file_entry[type]" "$upload_content_type";
                    upload_aggregate_form_field "file_entry[md5]" "$upload_file_md5";
                    upload_pass_form_field ".*";
                    upload_pass_args on;
                    upload_pass @unicorn_for_upload;
                    break;
                }

                if (!-f $request_filename) {
                    proxy_pass http://upstream_upload;
                    break;
                }
            }

            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
    }


## Testing through curl

    curl -XPUT localhost/files -F file=@localfile.txt
    curl -XPOST localhost/files -F file=@localfile.txt




