# BAD Node.js Dockerfile — run `/docker-emmet examples/node/bad` to see every finding.
# Each annotation names the rule it triggers (the DF-16 note explains one that intentionally does NOT).

FROM node:latest                          # DF-15 unpinned · DF-13 full image (not -slim/-alpine)

WORKDIR /app

ARG API_KEY                               # DF-01 secret in ARG
ENV DATABASE_PASSWORD=supersecret         # DF-01 secret in ENV

COPY . .                                  # DF-06 source copied before dependency install
RUN npm install                           # DF-17 npm install (not ci) · ships devDeps

RUN npm run build

RUN apt-get update && apt-get install -y curl   # DF-07 apt cache not cleaned · DF-09 no --no-install-recommends

# no USER instruction                     # DF-02 runs as root
# no .dockerignore in this dir            # DF-11

EXPOSE 3000
# node:latest is the FULL Debian image and DOES ship curl, so DF-16 correctly does NOT fire here.
HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1
CMD npm start                             # DF-04 shell form — sh becomes PID 1, signals lost
