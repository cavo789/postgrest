# URL rewriting

> [https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LI-H](https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LI-H)

Without URL rewriting, we need to write things like `http://localhost:3000/task?id=eq.1001` to get that task.

To be able to use `http://localhost:3000/task/1001`, we need to use some URLs rewriting.

For nginx:

```htaccess
# support /endpoint/:id url style
location ~ ^/([a-z_]+)/([0-9]+) {

  # make the response singular (see the `Don't return an array` chapter above)
  proxy_set_header Accept 'application/vnd.pgrst.object+json';

  proxy_pass http://localhost:3000/task?id=eq.$1;
}
```
