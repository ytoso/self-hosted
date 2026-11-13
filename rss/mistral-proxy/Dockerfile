# Bakes the nginx config into the image.
#
# Why: the config used to be bind-mounted from ./mistral-proxy into
# /etc/nginx/conf.d. When the container is (re)created before Docker Desktop has
# materialised that Windows path, the source is cached as an EMPTY DIRECTORY.
# nginx then starts fine (`nginx -t` passes, workers spawn) but has zero server
# blocks, so nothing listens on :80 and every request to http://mistral-proxy/rss
# fails - and `docker restart` does not re-resolve the bind, only `--force-recreate`
# does. Baking the file in removes the host-path dependency entirely.
FROM nginx:alpine

COPY default.conf /etc/nginx/conf.d/default.conf

# Fail the build if the config is unusable.
RUN nginx -t
