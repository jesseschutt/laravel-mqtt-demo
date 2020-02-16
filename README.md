# Laravel / MQTT Demo Application
For the entire story behind this setup please see the [article on my blog](https://jesseschutt.com/blog/integrating-mqtt-into-a-laravel-application) where I describe how we built a Laravel application that used MQTT to log temperature readings via simple sensors.

## Node Script
    process.title = 'mqtt-demo-process-node'
    
    const mqtt = require('mqtt')
    const axios = require('axios')
    const debug = process.env.NODE_ENV !== 'production'
    
    let endpoint = 'https://####.test/messages'
    
    if(process.env.NODE_ENV === 'production') {
        endpoint = 'https://####.com/messages'
    }
    
    if(debug){
        console.log('connecting')
    }
    
    const client = mqtt.connect('mqtt://m16.cloudmqtt.com:#####', {
        username: '####',
        password: '####'
    })
    
    client.on('connect', () => {
        if(debug) {
            console.log('connected')
        }
    
        client.subscribe('+/your-topic',{qos:1})
    })
    
    client.on('message',function(topic,message){
        if(debug) {
            console.log('this message :', message.toString());
        }
        axios.post(endpoint, {topic, message: message.toString()})
            .then(({ data }) => {
                if(debug) {
                    console.log(data);
                }
            })
            .catch(error => {
                if(debug) {
                    console.error(error);
                }
            });
    });
    
## Laravel Forge Configuration
We will need to stop the listener from time to time, so set up a daemon to keep `yarn mqtt-prod` running.

    /home/forge/your-site.com yarn mqtt-prod

### Deployment Script

Include `pkill -f mqtt-demo-process-node` in the deployment script so that the mqtt script will be restarted and pick up any new information when the supervisor restarts it.

    cd /home/forge/your-site.com
    git pull origin master
    composer install --no-interaction --prefer-dist --optimize-autoloader
    echo "" | sudo -S service php7.3-fpm reload
    
    yarn install
    
    yarn production
    
    # This value is set at the top of the subscriber.js
    pkill -f mqtt-demo-process-node 
    
    if [ -f artisan ]
    then
        php artisan migrate --force
    fi

### Artisan Command

This command allows the Laravel app to kill the node mqtt listener. Once the process is stopped the daemon should auto-restart `yarn mqtt-prod` and pick up any new connections.

    <?php
    
    namespace App\Console\Commands;
    
    use Illuminate\Console\Command;
    use Symfony\Component\Process\Process;
    
    class MQTT extends Command
    {
        protected $signature = 'mqtt:kill-process';
    
        protected $description = 'Stop the overseer mqtt node process';
    
        public function handle()
        {
            $process = new Process(['pkill', '-f', 'mqtt-demo-process-node']);
            $process->start();
    
            foreach ($process as $type => $data) {
                if ($process::OUT === $type) {
                    echo "\nRead from stdout: ".$data;
                } else {
                    echo "\nRead from stderr: ".$data;
                }
            }
        }
    }

