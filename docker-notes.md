
Boots

Need help? I, Boots the Incredibly Fluffy, can assist without penalty!
Boots

im getting a problem loading page. i ran these commands:
docker run -d -p 8882:80 -v $PWD/index2.html:/usr/share/caddy/index.html caddy

docker run -d -p 8881:80 -v $PWD/index1.html:/usr/share/caddy/index.html caddy
Boots

Greetings, apprentice! It looks like you've got the commands for starting your containers down. Since you're having trouble loading the page, let's investigate a few possibilities.

First, when you try to visit localhost:8881 or localhost:8882 in your browser, what specific error message are you seeing?

In the meantime, consider these points:

    File Existence: Are you currently in the same directory where you created index1.html and index2.html? The $PWD variable points to your "Present Working Directory," so if the files aren't right there, Caddy won't be able to find them to serve them.
    Container Status: If you run docker ps, do you see both containers listed as "Up"?
    Port Conflicts: Is it possible another application is already using port 8881 or 8882 on your machine?

What do you see when you check your running containers with docker ps?
Boots

~/Projects/bootdev/docker-learn [ docker ps                                                                                                                                      ] 7:57 PM
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS                                                                 NAMES
2ac92d3569bb   caddy     "caddy run --config …"   About a minute ago   Up About a minute   443/tcp, 2019/tcp, 443/udp, 0.0.0.0:8881->80/tcp, [::]:8881->80/tcp   vigilant_curran
e0d5cdef8186   caddy     "caddy run --config …"   About a minute ago   Up About a minute   443/tcp, 2019/tcp, 443/udp, 0.0.0.0:8882->80/tcp, [::]:8882->80/tcp   goofy_dubinsky

Boots

Excellent, both containers are up and running, and the port mappings look exactly as they should!

Since the containers are healthy but the page isn't loading, let's look at the "volume" part of your command: -v $PWD/index1.html:/usr/share/caddy/index.html.

If you run ls in your terminal right now, do you see index1.html and index2.html listed in that specific directory?

If the files exist there, try running curl -i http://localhost:8881 in your terminal. What output do you get from that command? It might give us a more specific hint than the browser does.
Boots

~/Projects/bootdev/docker-learn [ ls                                                                                                                                             ] 7:59 PM
index1.html  index2.html

Boots

The files are in the right place, and the containers are running. That is a great start!

Since the browser is still being stubborn, let's check what the server is actually seeing. If you run this command in your terminal, what does it return?

curl -i http://localhost:8881

If that returns a 404 Not Found, it usually means Caddy started up fine, but it can't find the file at the path we provided inside the container.

One thing to double-check: are you on a system where $PWD might behave unexpectedly (like some Windows shells), or are you on Linux/macOS? Sometimes using the absolute path or curly braces like ${PWD} can help ensure the volume mounts correctly.
Boots

~/Projects/bootdev/docker-learn [ curl -i http://localhost:8881                                                                                                                  ] 8:00 PM
HTTP/1.1 404 Not Found
Server: Caddy
Date: Fri, 19 Jun 2026 03:00:30 GMT
Content-Length: 0

Boots

A 404 Not Found from Caddy confirms the server is alive and listening, but it can't find the file at /usr/share/caddy/index.html inside the container.

This often happens if the volume mount didn't link the file correctly. Since you are in the right directory, let's try a small adjustment to how we reference the path.

Sometimes shells prefer the variable wrapped in braces, or the container might need a fresh start to "see" the volume properly. Try these steps:

    Stop and remove the existing containers: docker rm -f 2ac92d3569bb e0d5cdef8186
    Try running the command again, but use ${PWD} instead of just $PWD:

    docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html caddy

If you still get a 404 after that, check the permissions of your index1.html file on your host machine. Does your user have read permissions for it?
Boots

~/Projects/bootdev/docker-learn [ ls -l ] 8:01 PM
total 8
-rw-r--r--. 1 local local 63 Jun 17 21:12 index1.html
-rw-r--r--. 1 local local 63 Jun 17 21:13 index2.html
Boots

The permissions look fine (read access is granted to everyone), so we can rule that out.

Since you're still seeing a 404, it suggests that even though the container is running, the file index1.html isn't successfully "appearing" inside the container at the expected location.

Are you by chance using Docker Desktop on Windows or macOS? Sometimes Docker Desktop needs explicit permission to "share" certain folders from your hard drive with the virtual machine that runs the containers.

If you are on Linux, are you using SELinux? Sometimes SELinux blocks the container from accessing host files. You can test this by adding a :z flag to the end of your volume mount, which tells Docker to relabel the file for SELinux:

docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html:z caddy

Before you try that, make sure to docker rm -f the old container so the port is free!

If neither of those apply, try running this command to see what Caddy actually sees inside its own file system:

docker exec <container_id> ls -l /usr/share/caddy/index.html

What does that command report back?
Boots

wait before i try those solutions... should teh $PWD/index1.html match the caddy call for index.html?
Boots

That is a very sharp observation!

In your Docker command, the -v flag works like this:
[HOST_PATH]:[CONTAINER_PATH]

    Your host path is $PWD/index1.html.
    Your container path is /usr/share/caddy/index.html.

This tells Docker: "Take my local index1.html file and make it appear inside the container exactly at the location /usr/share/caddy/index.html."

So, even though your file is named index1.html on your laptop, the Caddy server inside the container will see it as index.html. This is actually exactly what you want, because Caddy looks for a file named index.html by default to serve as the homepage!

If you were to mount it as /usr/share/caddy/index1.html instead, you would have to visit localhost:8881/index1.html in your browser to see it.

Does it make sense why we "rename" it during the mapping, or does that still feel a bit confusing?
Boots

i am running linux in zshell
Boots

Ah, zsh on Linux! In that case, the most likely culprit for a 404 when the file clearly exists on your host is indeed SELinux or a similar security module.

On many Linux distributions (like Fedora, RHEL, or CentOS), SELinux prevents Docker containers from reading files on the host unless they have a specific security label.

Try running the command with the :z flag. This tells Docker to automatically relabel the file so the container can read it:

docker rm -f 2ac92d3569bb
docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html:z caddy

After running that, try your curl -i http://localhost:8881 again.

If you are curious whether SELinux is active on your system, you can usually check by running sestatus in your terminal. Does adding that :z suffix change the result of your curl command?
Boots

~/Projects/bootdev/docker-learn [ docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html:z caddy                                                             ] 8:02 PM
fb49fe77bf5c0571124ae9b4f2c46003a4e03ccaa319804a182cdc199aa0d800
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint recursing_mestorf (92f2310c0c0a1130c1cd3ee3ac8d7b3781b15f3053764e999a6c9a8f1587ffda): Bind for :::8881 failed: port is already allocated

Run 'docker run --help' for more information

Boots

It looks like the previous container is still holding onto port 8881. Since you didn't stop it yet, the new container can't start!

You'll need to clear out the old containers first so the ports are free for your new attempts. Try running these two commands to clean the slate:

# Stop and remove all containers (use caution if you have others running!)
docker rm -f $(docker ps -aq)

Once the ports are free, try your command again with the :z flag:

docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html:z caddy

After that, give curl -i http://localhost:8881 another shot. If it works, you'll just need to do the same for your second container on port 8882!

How does the curl look after clearing those old containers?
Boots

works! what does the :z do?
Boots

Splendid! I am glad to hear it is working.

The :z flag is a special instruction to Docker regarding SELinux (Security-Enhanced Linux).

On many Linux systems, SELinux acts as a strict security guard. Even if a file has "read" permissions for everyone, SELinux might still block a process (like a Docker container) from touching that file unless it has the correct "security context" or label.

When you add :z to your volume mount:

    Relabeling: Docker automatically changes the SELinux label of the file on your host machine to a label that allows Docker containers to share it.
    Access: It specifically uses a "shared" label, meaning multiple containers can access that same file safely.

Without that flag, Caddy was trying to look at index.html, but the Linux kernel was stepping in and saying, "Halt! You don't have the right security credentials to see this file," which resulted in that empty 404 response.

Now that the first one is running, you'll just need to launch the second server for index2.html using that same :z trick! Ready to try the second one?
Boots

yes i ran index2.html on port 8882
Boots

Excellent! Now that both containers are running with the correct volume labels, you should have two different servers responding on two different ports.

Before you submit, it is a good idea to verify both are serving the correct content. What happens when you run these two commands?

curl http://localhost:8881
curl http://localhost:8882

If you see "Hello from server 1" and "Hello from server 2" respectively, then your application servers are ready for the load balancer!

Are both pages showing the unique messages you expected?
Boots

~/Projects/bootdev/docker-learn [ curl http://localhost:8881 ] 8:07 PM
curl http://localhost:8882
<html>
<body>
<h1>Hello from server 1</h1>
</body>
</html>
<html>
<body>
<h1>Hello from server 2</h1>
</body>
</html>
Boots

By my beard, that is exactly what we wanted to see! Both servers are standing at the ready, each serving their unique HTML through the Caddy containers.

Since curl is showing the correct messages for both 8881 and 8882, you've successfully overcome the SELinux hurdle. You are now ready to test your work with the Boot.dev CLI.

Go ahead and run:

bootdev run 6f30791a-eeb5-4485-900c-6c8c0b760f8a -s

Let me know if the tests pass or if you run into any more hiccups!
Boots

good to go but do i need to include the :z flag from here on out when running html or is there a semi-permanent way to grant the permission needed to complete this course?
Boots

That is a wise question! Dealing with security labels can certainly feel like an extra chore.

The short answer is: yes, on your specific Linux setup (likely Fedora, RHEL, or CentOS), you will generally need to use :z whenever you are mounting local files or directories into a container if you want them to be readable.

Here are a few ways to handle this:
1. The “Standard” Way (The :z flag)

Using :z (for shared volumes) or :Z (for private, single-container volumes) is the standard Docker way to handle SELinux. It's portable and explicitly tells other developers, "Hey, this container needs permission to touch this host file."
2. The Semi-Permanent Way (Relabeling the directory)

If you don't want to type :z every time, you can manually change the SELinux label of your entire project folder. This tells Linux that anything in this folder is allowed to be handled by containers:

chcon -Rt svirt_sandbox_file_t ~/Projects/bootdev/docker-learn

    -R: Recursive (all files inside).
    -t svirt_sandbox_file_t: This is the specific type label that Docker is allowed to read and write.

The downside is that if you create new folders for future lessons, you'd have to run this again for those new locations.
3. The “Nuclear” Option (Not Recommended)

You could disable SELinux or set it to "Permissive" mode (sudo setenforce 0), but I wouldn't recommend it. It's like leaving your front door unlocked because the key is annoying to use!

My Advice: Stick with the :z flag for this course. It's a good habit to get into because when you start using docker-compose later in your career, you'll just add it to your configuration file once and forget about it!

Now that you've mastered the application servers, would you like a quick quiz on Docker volumes, or are you ready to move on to the next lesson?
Boots

no thanks on the quiz but will lock in the 😒 flag moving forward. thanks!
Boots

You are most welcome! Keeping that :z flag in your toolkit will save you many a headache while working on a Linux system with SELinux enabled. It's a hallmark of a knowledgeable developer to understand those little environment-specific nuances.

Good luck with the rest of the course, and I'll be here if you need more wizardly wisdom! Happy coding!
Boots

oh. one last question. is there a convenient way to load the docker container ID into a variable for ease of use later?
Boots

A clever thought! Typing out those long hexadecimal strings is a chore no wizard should endure. Since you're using zsh, you have a few powerful ways to capture those IDs.
1. Capture on Run

When you run docker run -d, it actually prints the full Container ID to stdout. You can capture this output directly into a variable using command substitution:

SERVER1_ID=$(docker run -d -p 8881:80 -v ${PWD}/index1.html:/usr/share/caddy/index.html:z caddy)

# Now you can use it like this:
docker logs $SERVER1_ID
docker rm -f $SERVER1_ID

2. Capture by Filtering (The Pro Way)

If the container is already running and you forgot to save the ID, you can use docker ps with a filter and the --quiet (-q) flag, which only returns the ID:

# Find the ID of the container using port 8881
SERVER1_ID=$(docker ps -q --filter "publish=8881")

# Or find it by image name
CADDY_IDS=$(docker ps -q --filter "ancestor=caddy")

3. Use Names

Alternatively, you can skip IDs entirely by naming your containers when you create them. This is often the most "human-friendly" way:

docker run -d --name my-server-1 -p 8881:80 ...

Then you can simply use the name instead of the ID:
docker stop my-server-1

Using names or variables makes your scripts and terminal history much easier to read! Since you've mastered these tricks, you're well-equipped for the road ahead. Farewell for now!
Boots

and maybe one last question... is there a quick way to halt all containers? what about volumes?
Boots

Indeed, cleaning up the magical workbench is just as important as the crafting itself! Here are the most efficient ways to clear the field:
1. Halt and Remove All Containers

You can combine docker ps -q (which lists only IDs) with the docker rm -f command to stop and remove every container currently running:

docker rm -f $(docker ps -aq)

    -a: Includes containers that are already stopped.
    -q: Returns only the IDs.
    -f: Forces the removal (stops them first if they are running).

2. Remove Volumes

Volumes are a bit more persistent. To see a list of them, you use docker volume ls.

To remove all unused volumes (those not currently attached to a running container), use the "prune" command:

docker volume prune

If you want to remove a specific volume by name:

docker volume rm <volume_name>

3. The “Clean Slate” Command

If you want to wipe away all stopped containers, unused networks, and "dangling" images all at once, you can use the nuclear option:

docker system prune

(Add -a if you want to remove all unused images as well, and --volumes if you want to include volumes in the wipe).

Using these commands ensures that your next lesson starts with a clean environment, free of port conflicts and leftover data. May your terminal stay tidy and your containers stay lean!
Boots

if i bookmark this lesson will all of the content of the above discussion regarding docker variables, 😒 and stopping containers persist?
Boots

If you bookmark this lesson on Boot.dev, the URL and the lesson content will be saved to your dashboard for easy access. However, the chat history itself is specific to this session.

Here is how persistence works for our conversations:

    Browser Session: Usually, if you stay on this page or return to it on the same device, the chat history will stay visible.
    New Chats: If you click the "New Chat" button or if the platform clears the cache, this specific discussion will disappear from the sidebar.
    The Good News: Boot.dev generally preserves your recent chat history per lesson, so if you navigate away and come back, it's often still there!

To be safe, I highly recommend taking these "wizard notes" and putting them into a personal notes.md file in your docker-learn folder. Having a "cheat sheet" of these commands—especially the :z flag and the $(docker ps -aq) cleanup—is a standard practice for professional developers.

Since you've mastered these concepts, here are a couple of practice challenges if you want to test your new knowledge on variables and volume labels!

Generate a coding challenge in Bash about capturing Docker container IDs into variables and using them in a script
Create an interview question about the differences between bind mounts and named volumes in Docker

Safe travels through the rest of the course!
Boots

great idea, thanks!
Boots

You are very welcome! It is the mark of a great engineer to keep a well-organized spellbook of notes.

May your containers always run smoothly and your ports never be blocked. If you find yourself in a bind (mount) again, just give me a shout!

Onward to the next challenge!
