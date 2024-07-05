# Installation & Configuration

As the `root` user or as an user with root permissions, perform the following actions
### Utilities
##### Neovim

```bash
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:neovim-ppa/stable
sudo apt-get update
sudo apt-get install neovim
sudo apt-get install python-pip python3-dev python3-pip
sudo apt install python3-neovim
sudo apt-get install zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
chsh -s $(which zsh)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```

Add this to your `.zshrc` file:
```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

Then:
```bash
source ~/.zshrc
nvm install --lts
curl -fsSL https://get.pnpm.io/install.sh | sh -
pnpm install pm2@latest -g
sudo apt install git-all
```

#### MongoDB
View the official MongoDB installation guide for Ubuntu 20.04 [here](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/#std-label-install-mdb-community-ubuntu).

#### Certbot
Certbot is used to generate SSL certificates for the server.
View the official Certbot installation guide for Ubuntu 20.04 [here](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal).
The version of this server may be different and that should be fine (Ubuntu 22, 24).

#### Nginx
Nginx is installed by default but you will want to install the certbot plugin for it.
```bash
sudo apt-get install python3-certbot-nginx
```

### File structure

```
/root
    /projects   -- Projects are located here.
    /scripts    -- Utility scripts for installation, configuration and operations.
    /README.md  -- How to manage the server and useful information.
```

## SSH Connection

The server uses SSH keys to connect, so it is necessary to generate a key pair and add the public key to the server.
The `root` user still exists, but it is not recommended to use it for security reasons.

You might want to use `ssh-agent` & `ssh-add` to manage the keys, connect to the server easily, and use your `git` account.


### Adding a public SSH key to the server

If you don't have an SSH key, generate yours with the following command:
```bash
ssh-keygen -t ed25519 -C "<comment about what key is this from>"
```

From your local machine and with you generated `id_ed25519.pub` (or `id_rsa.pub`) key:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<server_ip>
```

### Forwards Agent
```bash
ssh -A root@<server_ip>
```

You can also add the server in your `~/.ssh/config` file:
```bash
Host <name_the_server>
    Hostname <server_ip>
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
```

#### Using zsh locally
If using zsh locally, you may want to add the following to your `~/.zshrc` file to automatically handle `ssh-agent`:
```bash
plugins=(git ssh-agent)

zstyle :omz:plugins:ssh-agent agent-forwarding yes

# Mac OS X only
zstyle :omz:plugins:ssh-agent ssh-add-args --apple-load-keychain
```

## Firewall

To view open ports:
```bash
sudo ufw status
```
The ports include: 
```
Nginx Full                 ALLOW       Anywhere 
22                         ALLOW       Anywhere
443                        ALLOW       Anywhere
```


Less is better, but you can add more ports if needed.

Note that to use MongoDB from the outside, you need to open the port `27017` to a specific IP only (for security reasons).
You can then test the port connection with the following command:
```bash
nc -zv <current server ip address> 27017
```


# Adding & Managing a Project

Each project has three main parts to handle:
1. the project code/build itself;
2. the databases;
3. the server configuration (reverse proxy and hosting);

## Project Code / Build

### Cloning and pulling the project
When adding a project, you need to `git clone` the repository inside the `/root/projects` directory.
To update a project already cloned, you can `git fetch` & `git pull` the repository.

Once a project is on the server, you can build it and install its dependencies:

This usually involves running the following commands inside the directory of the project:
```bash
pnpm install
pnpm build
```

### Running the project
To run the project, you can use `pm2` to start the process:
```bash
pm2 start <command> --name <processName> --max-memory-restart <3000MB> --log <log_path>
```

You can then manage the active processes with the following commands:
```bash
pm2 restart <processName>
pm2 reload <processName>
pm2 stop <processName>
pm2 delete <processName>

pm2 monit
pm2 list
pm2 logs --lines 200

pm2 save        #Save an ecosystem
pm2 resurrect   # Resurrect an ecosystem
```

### Environment Variables
Make sure not to forget the `.env` or `.env.local` file with the necessary environment variables for the project to run.


## Database(s)

By default, the server uses MongoDB, but you can use any other database system.

### MongoDB

Each project should have its own MongoDB database and user. There is a global user for the server that is created with:
```bash
mongosh
use admin
db.createUser({user: "admin", pwd: "<password>", roles: ["root"]})
```
Save those credentials in a safe place.

To create a new database and user for a project, you can use the following commands:
```bash
mongosh
use <project_name>
db.createUser({user: "<project_name>", pwd: "<password>", roles: ["dbOwner"]})
```

You can then use a connection string to connect to the database:
```bash
mongodb://<project_name>:<password>@127.0.0.1:27017/<project_name>
```

If trying to access the database from the outside, you need to open the port `27017` to a specific IP only (for security reasons).
You also have to add the IP to the MongoDB configuration file:
```bash
sudo nano /etc/mongod.conf

# Add the following line:
net:
  bindIp: 127.0.0.1,<your_server_ip>
```

### Database Backup
_TODO: verify if that backup is complete and can be restored._
You can use the `mongodump` and `mongorestore` commands to backup and restore databases.


## Hosting & Reverse Proxy

### Nginx
The server uses Nginx to host the projects and manage the reverse proxy.
All the projects should run of different ports from 3001 to 30.. in order for facilitating the reverse proxy configuration.

To add a new project, you need to create a new configuration file in the `/etc/nginx/sites-available` directory.

For instance, to add the project `mywebsite.com`, you can create a file named `mywebsite.com` with the following content:
```nginx
server {
    listen 80;
    server_name mywebsite.com www.mywebsite.com;

    location / {
        proxy_pass http://localhost:30..;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

You can then create a symbolic link to the `/etc/nginx/sites-enabled` directory to enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/mywebsite.com /etc/nginx/sites-enabled/
```

Make sure to replace the `mywebsite.com` and `30..` with the actual project name and port.
You can then check that you configuration files are valid valid with:
```bash
sudo nginx -t
```

And reload the Nginx configuration with to active the new site:
```bash
sudo systemctl reload nginx
```

To delete a site, you can remove the symbolic link and delete the configuration file:
```bash
sudo rm /etc/nginx/sites-enabled/mywebsite.com
# Optionally, if you don't need it anymore
sudo rm /etc/nginx/sites-available/mywebsite.com
```

### Certbot

Before adding a certificate to a new project, you need to make sure that the domain is pointing to the server's IP address.

To add an SSL certificate to a site, you can use the following command:
```bash
sudo certbot --nginx -d mywebsite.com -d www.mywebsite.com
```

You can then check the certificate with:
```bash
sudo certbot certificates
```

You can also renew the certificates with:
```bash
sudo certbot renew
```

# Backup & Recovery

## MongoDB
You can use the `mongodump` and `mongorestore` commands to backup and restore databases.
If you lost access to all databases, you can set the `disabled` flag to `true` in the `/etc/mongod.conf` file and restart the MongoDB service.:
```bash
sudo nano /etc/mongod.conf
```

```yaml
security:
  authorization: "disabled" # instead of "enabled"
```
Restart the MongoDB service:
```bash
sudo systemctl restart mongod
```

You can then access the databases and users to recreate them.
__Don't forget to set the `authorization` flag back to `enabled` and restart the service!__

## Recover server root access
Use the _DigitalOcean_ admin panel for the droplet in the _Access_ tab:
