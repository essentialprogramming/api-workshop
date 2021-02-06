
### Deploy Prisma to Digital Ocean
   ###### This guide will assume you already have a DigitalOcean account
    
   1. Install Prisma on your Droplet  
  
       * Connect the Digital Ocean Droplet set up in the previous step:  
       ```ssh root@__IP_ADDRESS__```  
       Adding our Droplet's IP address this looks like the following:  
      ```ssh root@37.138.15.166 -i /Users/.ssh/id_rsa_do```  
      
       * You need to install both Docker and Docker Compose. DigitalOcean has detailed guides for installation that you can find here and here.  
         For the quick path, execute these commands on your Droplet:  
         ```
         curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
         sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
         sudo apt-get update
         sudo apt-get install -y docker-ce
         sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
         ```
       * Install Node.JS:  
          ```
          curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
          sudo apt-get install -y nodejs
          ```
       * Install the Prisma CLI:  
         ```
         npm -g install prisma
         ```  
        
           ###### The Prisma CLI is now installed on your DigitalOcean Droplet. Next, you need to configure and start the Prisma server.
       
   2. Start the Prisma server 
        ###### In this section you're going to setup the infrastructure needed to deploy the Prisma service to the Droplet.
        * In your Droplet, create a docker-compose.yml file
            ```
            touch docker-compose.yml
            ```
        * In this file, add the configuration below :    
          ```
          services: 
            mysql: 
              environment: 
                MYSQL_ROOT_PASSWORD: prisma
              image: "mysql:5.7"
              restart: always
              volumes: 
                - "mysql:/var/lib/mysql"
            prisma: 
              environment: 
                PRISMA_CONFIG: |
                    port: 4466
                    managementApiSecret: my-secret
                    databases:
                      default:
                        connector: mysql
                        host: mysql
                        port: 3306
                        user: root
                        password: prisma
                        migrations: true
              image: "prismagraphql/prisma:1.34"
              ports: 
                - "4466:4466"
              restart: always
          version: "3"
          volumes: 
            mysql: ~
          ```  
          > The managementApiSecret is used to secure your Prisma server. It is specified in the server's Docker configuration and later used by the Prisma CLI to authenticate its requests against the server.
         * Now run the following command in your terminal:  
            ```
            docker-compose up -d
            ```
         * This will fetch the Docker images for both prisma and mysql. To verify that the Docker containers are running, run the following command:  
            ``` docker ps```  
            
            Expected output :  
            
            ```
            CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                    NAMES
            24f4dd6222b1        prismagraphql/prisma:1.34   "/bin/sh -c /app/sta…"   15 seconds ago      Up 1 second         0.0.0.0:4466->4466/tcp   root_prisma_1
            d8cc3a393a9f        mysql:5.7                   "docker-entrypoint.s…"   15 seconds ago      Up 13 seconds       3306/tcp                 root_mysql_1
             ```
   3. Deploy the Service  
   
        Now that your Prisma server and its database are running via Docker, you will deploy the Prisma service.
        We are going to bootstrap a new Prisma service via   ```prisma init```.
   
        * On your local machine ,run the following command in your terminal:  
            ```
            prisma init hello-world
            ```
        * Following the interactive prompt :  
            1. Select *Use other server*
            2. Enter the IP of the DigitalOcean Droplet  
                > Example :  http://157.230.10.243:4466  
                The IP must be this format (specify the protocol and the port that was previously set up for Prisma in the .yaml file)  
            
            3. Enter the secret that was previously set up for Prisma in the .yaml file: 
            ```my-secret```  
            4. Choose a name for your service  
                > By pressing enter you choose the default name which is ```hell-world```  
            
            5. Choose a name for your stage 
                > By pressing enter you choose the default name which is ```dev```
            6. Select the programming language for the generated Prisma client  
                > You can choose any desired programming language for the to be generated Prisma client.  
             
             * This command should output:  
                 ```
                   Created 3 new files:
                   
                   prisma.yml           Prisma service definition
                   datamodel.graphql    GraphQL SDL-based datamodel (foundation for database)
                   .env                 Env file including PRISMA_API_MANAGEMENT_SECRET
                 ```
         * Navigate into the hello-world directory that was generated from ```prisma init``` and deploy your Prisma service with the following commands:  
         ```cd hello-world```  
         ```prisma deploy```
            * This command should output :  
                ```Creating stage default for service default ✔
                   Deploying service `default` to stage `default` to server `default` 653ms
                   
                   Changes:
                   
                     User (Type)
                     + Created type `User`
                     + Created field `id` of type `GraphQLID!`
                     + Created field `name` of type `String!`
                   
                   Applying changes 1.2s
                   
                   Your Prisma endpoint is live:
                   
                     HTTP:  http://37.139.15.166:4466/hello-world/dev
                     WS:    ws://37.139.15.166:4466/hello-world/dev
                   
                   You can view & edit your data here:
                   
                     Prisma Admin: http://37.139.15.166:4466/hello-world/dev/_admin
               ```
        Connect to the /management endpoint in a browser. For example, if your Droplet has the IP address of ```37.139.15.166``` open the following webpage: http://37.139.15.166:4466/management. If a GraphQL Playground shows up, your Prisma server is set up correctly.
                   
        Finally, connect to the / endpoint in a browser. For example, if your Droplet has the IP address of ```37.139.15.166``` open the following webpage: http://37.139.15.166:4466. You can now explore the GraphQL API of your Prisma service.
                   
        You can send the following mutation in GraphQL Playground to create a new user:
        ```
         mutation {
           createUser(data: { name: "Alice" }) {
             id
             name
           }
         }
         ```
        With the newly created User, run a query by it's id:  
        ```
        query {
          user(where: { id: "cjkar2d62000k0847xuh4g70o" }) {
            id
            name
          }
        }
        ```
   4. Create your own schema  
        * The GraphQL schema resides inside ```hello-world``` directory in a file called ```datamodel.prisma```
        * To modify the schema we will use ```vi``` text editor. Follow these steps in order to have a smooth experience with ```vi``` :  
            *  in terminal type : ```vi datamodel.prisma```
            *  go to *command mode* in the editor by pressing ESC key on the keyboard.
            *  type :  * ```:%d``` to delete all lines from that specific file.
            *  copy the following text and paste it in ```vi``` :  
                ```
                type Book {
                  id : ID! @id
                  title: String
                  author: Author
                }
                
                type Author {
                  id : ID! @id
                  name: String
                  books: [Book]
                } 
                ```
            * in order to save & exit press ```Esc``` and type ```:wq```  
                > If you want to exit without saving from vi press ```Esc``` and type ```:q!```
        
        * All we have to do to re-deploy our schema is to enter the following command used previously :  
            ```prisma deploy```
        * If everything works as expected you should receive the following output :  
            ```Changes:
            
              User (Type)
              - Deleted type `User`
            
              Book (Type)
              + Created type `Book`
              + Created field `id` of type `ID!`
              + Created field `title` of type `String`
              + Created field `author` of type `Author`
            
              Author (Type)
              + Created type `Author`
              + Created field `id` of type `ID!`
              + Created field `name` of type `String`
              + Created field `books` of type `[Book!]!`
            
              AuthorToBook (Relation)
              + Created an inline relation between `Author` and `Book` in the column `author` of table `Book`
            
            Applying changes 1.1s
            Generating schema 37ms
            Saving Prisma Client (TypeScript) at /root/hello-world/generated/prisma-client/
            
            Your Prisma endpoint is live:
            
              HTTP:  http://157.230.10.243:4466/hello-world/dev
              WS:    ws://157.230.10.243:4466/hello-world/dev
            
            You can view & edit your data here:
            
              Prisma Admin: http://157.230.10.243:4466/hello-world/dev/_admin
            ```
        
           