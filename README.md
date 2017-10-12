This example docker compose file sets up two docker registries on a single docker host (or swarm, if your machine is set up that way). They provide a pull-through mirror of Docker Hub (pull-only) that allows you to reduce the amount of pulls that have to go out to the internet, and a writable (push-pull) local registry for storing your own images locally.

This particular configuration adds a slightly risky but quite handy tweak: everything you push to the local registry will be available for pulling from the mirror registry. When you set the mirror as the default registry, the most common use case (docker pull) never needs to specify which registry it's pulling from.

# Example use case.

Say I want to make a local variant of the popular alpine image and publish internally for my colleages to use. First, I need to make my local variant:

```sh
sudo docker pull alpine
./customize_image alpine local_alpine
```

The `customize_image` script has made the expected changes and created a new image local_alpine. Now I want to push local_alpine to a local registry that my colleagues can pull it from. Assuming my registry is at docker.registry.local:5000, I can tag and push the image thus:

```sh
sudo docker tag local_alpine docker.registry.local:5000/local_alpine
sudo docker push docker.registry.local:5000/local_alpine
```

Slightly wordy, but all this belongs in a CI pipeline somewhere, so you only have to figure it out once. The bit that happens much more frequently is when anyone pull this CI-built image for their use:

```sh
sudo docker pull local_alpine
```

Which of course gives us

```
Using default tag: latest
Error response from daemon: pull access denied for local_alpine, repository does not exist or may require 'docker login'
```

Oops, wrong registry. Actually, we need to do this:

```sh
sudo docker pull docker.registry.local:5000/local_alpine
```

And we need to remember when to pull from the local registry and when to pull from the default public registry (Docker Hub).

# I want to just `docker pull <image>`, alright?

Sure thing. The lovely people at Docker know that this is a requested feature, and are thinking about the best way to solve it without risking pulling the wrong image, pushing the right image to the wrong place, or falling victim to image identity fraud. But for now, I have a hack workaround.

The trick is to share a single registry location with two different registries, one pull-through mirror of Docker Hub and one push-pull local registry. This solves my immediate problem, but be aware, it comes with a risk. If both registries write to the same file at the same time, you'll probably have two corrupted registries. There's no guaranteed way to avoid this, short of shutting down one registry every time you perform a write action of any sort on the other. If it ever happens to me (it hasn't yet), then I plan on blowing away and rebuilding my registry. This is very low cost for me since I have very few internally-published images, but if you've got lots of images and don't want to risk losing your registry, then this hack is not for you.


# Step 1. Set up your push-pull local registry.

Create a new directory for your docker configuration code. The official explanation of the technology is [on Docker docs](https://docs.docker.com/registry/deploying/). In this example, I'll be putting the shared registry directory in a subdirectory called `cache`. You can put it anywhere on your docker host that docker can bind mount it from.

I grabbed the config.yml from the official registry:2 image just to be explicit and to make future changes easier.

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
delete:
  enabled: true
```
The `delete: enabled: true` part lets docker do some occasional house-cleaning. This is where most of the risk of corruption comes from. Don't have this in both config files; it may be worth removing it from this file, and instead occasionally blowing the entire registry away and starting from scratch.

I'm deploying the registry via a compose file so that I can add my other registry in the next step. You don't have to use docker-compose, but if you do, this is my example file:

```yaml
version: '3.3'

# Docker-compose file for a local docker registry.
services:
  local_store:
    image: registry:2
    ports:
      - 127.0.0.1:5000:5000
    volumes:
      - ./cache:/var/lib/registry
      - ./store.yml:/etc/docker/registry/config.yml
```
This assumes that the config file above is called `store.yml`. The local `cache` directory will be bind mounted as the directory the registry stores all persistent data in: this is the line shared with the other registry in the next step.

# Step 2. Set up your pull-through local registry mirror of Docker Hub.

There's a recipe for this [on Docker docs](https://docs.docker.com/registry/recipes/mirror/), but in essence you just need another local registry in your compose file:

```yaml
  local_cache:
    image: registry:2
    ports:
      - 127.0.0.1:5001:5000
    volumes:
      - ./cache:/var/lib/registry
      - ./cache.yml:/etc/docker/registry/config.yml
```
The `/var/lib/registry` bind mount is identical to the other registry in step 1. The `cache.yml` here is a similar config file as before, tweaked according to the recipe:

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
proxy:
  remoteurl: https://registry-1.docker.io
# No username or password. Only public access to the public registry.
```

# Step 3. Deploy your registries.

Assuming you saved your docker compose file as registry.yml then you can do this if you're not using Swarm:

```sh
sudo docker-compose -f registry.yml up
# And you'll use "sudo docker-compose -f registry.yml down" to turn off the registries, if you need to.
```

Or this if you are using Swarm:

```sh
sudo docker stack deploy -c registry.yml registry
# And you'll use "sudo docker stack delete registry" to turn them off.
```

# Step 4. Use your registries.

On each docker host (machine running the docker daemon), you'll need to configure the daemon to use those registries. You can do this using command line parameters, but I prefer editing `/etc/docker/daemon.json`.

```json
{
    "registry-mirrors": ["http://docker.registry.local:5001"],
    "insecure-registries": ["docker.registry.local:5000", "docker.registry.local:5001"]
}
```

Restart each daemon in order to pick up those changes.

```sh
sudo systemctl restart docker
```

# Step 5. Use in anger.

Now the original use case works. Having deployed an image to the local registry:

```sh
sudo docker pull alpine
./customize_image alpine local_alpine
sudo docker tag local_alpine docker.registry.local:5000/local_alpine
sudo docker push docker.registry.local:5000/local_alpine
```

Remember the broken step from the introduction?

```sh
sudo docker pull local_alpine
```

Now it works! Any image will be pulled from the local registry set up as the docker mirror, as configured by "registry-mirrors" above. If the image has previously been pushed there or been cached after being pulled from Docker hub, then it is immediately pulled from the local registry. Otherwise it is pulled from Docker Hub and cached, ready for next time. 
