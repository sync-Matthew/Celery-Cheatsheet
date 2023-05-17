# Celery-Cheatsheet
How to: Add Celery to a Django project. Development and Production versions.

<h2>Creating Celery Services - Development</h2>  

<h3>Setup</h3>  

[Celery Docs for Django](https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html#django-first-steps)  

[Other useful tutorial](https://realpython.com/asynchronous-tasks-with-django-and-celery/)  

1. Install dependencies to your virtual environment:  
`pip install celery`  
`pip install redis`  

2. Install Redis:  
`sudo apt update`  
`sudo apt install redis`  

3. From `your_project/your_app` create a new file called `celery.py` and paste the following contents:  
```
import os

from celery import Celery

# Set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_app.settings')

app = Celery('your_app')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django apps.
app.autodiscover_tasks()
```  
*Remember to replace 'your_app' with the name of your applicaiton*  

4. Add your Redis broker to `your_app/settings.py`:  

```
# your_app/settings.py

# ...

# Celery settings
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
```  

5. Edit `your_app/init.py`:  

```
# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)
```  

6. Start your Django server, the Redis server, and Celery from separate terminals:  
`python manage.py runserver`  
`redis-server`  
`celery -A your_app worker -l INFO`  

<h3>Create Tasks</h3>  

1. From `some_other_app`, create a new file called `tasks.py`:  

```
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```  

2. From `some_other_app/views.py`, import and use your new task:  

```
from .tasks import add

def run_task(request):
    add.delay(1, 2)
```  

3. Restart Celery (`ctrl+c`) and `celery -A your_app worker -l INFO`  

The output should look something like this:  
![image](https://github.com/sync-Matthew/Faster/assets/109091963/2c61b9f5-c51d-4adf-bbeb-af8dbf2caf4a)  

<h2>Creating Celery Services - Production</h2>  

1. Install packages to the server's virtual environment where your project is hosted:  

`pip install celery`  
`pip install redis`  

2. [Add Redis Server to DO Ubuntu 22.04 Droplet](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-redis-on-ubuntu-22-04)  

`sudo apt update`  
`sudo apt install redis-server`  

From `sudo nano /etc/redis/redis.conf` find the supervised directive. Change it from ___ to `supervised systemd`:  
```
. . .

# If you run Redis from upstart or systemd, Redis can interact with your
# supervision tree. Options:
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
<mark>supervised systemd</mark>

. . .
```  

Afterwards, restart the Redis service:  
`sudo systemctl restart redis.service`  

Check the status of Redis with:  
`sudo systemctl status redis`  

3. Create a Celery service.  

From `/etc/systemd/system/` run the command:  
`touch celery.service`  

Paste the following contents into `/etc/systemd/system/celery.service`:  

```
[Unit]
Description=Celery Service
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/home/your_project/server
ExecStart=/home/your_project/.venv/bin/celery -A faster worker -l INFO
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=celery

[Install]
WantedBy=multi-user.target
```  

*Remember to replace WorkingDirectory and ExecStart values to match your project structure*  

After adding your celery service file, run the following commands to enable and start it:  
`sudo systemctl daemon-reload`  
`sudo systemctl enable celery`  
`sudo systemctl start celery`  

To check the status of your celery service, run the following:  
`sudo systemctl status celery`  

It should look something like this:  

![image](https://github.com/sync-Matthew/Faster/assets/109091963/9bf8ac7c-868d-4e52-a72b-4d72f1e57643)  
