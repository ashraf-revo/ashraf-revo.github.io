# Host Routing part 3
previously we discussed host mapping , and we also make it to load user info based on domain

now we will continue on this path and make this project to load certificates based every request url domain

### How we can configure our server to serve unlimited number of  certificates not just one or few numbers

we can customize our `reactor-netty` server to load certificates from a supplier function


