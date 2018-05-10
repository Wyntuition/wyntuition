# NGINX Production setup via Docker

here's an example of nginx configured as a reverse proxy production-ready with docker, https://github.com/evertramos/docker-compose-letsencrypt-nginx-proxy-companion

which uses the following repo which generates nginx reverse proxies in docker, https://github.com/jwilder/nginx-proxy 

## Tuning

- Establish a baseline of where we're at now probably based on requests/sec with the load testing tool [ab](https://httpd.apache.org/docs/2.4/programs/ab.html).  `ab -c 300 -n 50000 https://save-me` uses this tool to keep 300 connections open until it hits 50,000 requests. A fuzzy median value for default nginx settings gets around 3000 requests/sec.

ab is in apache-utils, `apt-get install apache2-utils`
```bash
FROM ubuntu
RUN apt-get update -y && apt-get install apache2-utils
```

- Set worker threads to auto which tells NGINX to create one worker thread for each CPU available to the system. For most systems, one worker process per CPU is an even balance of performance and reduced overhead. However, when serving static content we might bump it up, starting at 2x the number of CPUs and increasing by 4 to see if requests/sec increase significantly. Run `ab -c 300 -n 50000 https://save-me` again and see if there's a significant difference.
- check the nginx `open_file_cache_valid` parameter. This is how long static files will be cached open on the nginx server. So if it's say 10, if no one accesses them for 10 seconds they're closed. Going from say 10 to 120s can significantly reduce the number of times that NGINX must open and close static HTML files.
- disable accept_mutex, enable sendfile, set sendfile_max_chunk which can reduce the maximum time spent in blocking sendfile() calls, since NGINX won’t try to send the whole file at once, but will do it in 512k chunks.

More info, https://blog.codeship.com/tuning-nginx/

- tuning nginx's processing queues (via worker threads) that got them 9x performance, https://www.nginx.com/blog/thread-pools-boost-performance-9x/

  1. It requires a data set that is guaranteed not to fit in memory. On a machine with 48 GB of RAM, we have generated 256 GB of random data in 4‑MB files, and then have configured NGINX 1.9.0 to serve it.

- Run in parallel, cat /mnt/urls/urls | parallel --colsep ',' "ab -e {2} -c 2 -n 200 {1} && cat {2} | tail -n 100 | sed 's/^/{2},/' >> /tmp/ab.csv"`
