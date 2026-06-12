# BAD Node image — full base, a baked-in secret, runs as root, no healthcheck.
# `/docker-emmet examples/node/bad` reports this alongside the other scenarios + the compose file.

FROM node:latest                          # DF-13 full image (use -slim/-alpine) · DF-15 unpinned (`latest`)
WORKDIR /app
ENV DATABASE_PASSWORD=supersecret         # DF-01 secret in ENV — baked into every image layer
COPY . .                                  # DF-06 source copied before install — busts the npm cache
RUN npm install                           # DF-17 use `npm ci` for reproducible installs
RUN npm run build
# no USER                                 # DF-02 runs as root
# no HEALTHCHECK                          # DF-22 long-running server with no healthcheck
# no .dockerignore in this dir            # DF-11
EXPOSE 3000
CMD npm start                             # DF-04 shell form — sh becomes PID 1, SIGTERM is lost
