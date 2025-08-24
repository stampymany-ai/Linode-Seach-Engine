# Linode Seach Engine
A project made to help user make a search engine on linode
# The Guide Requirments
This guide requies you to have a Linode account and to have a created Ubuntu 24.04 LTS VM.
# Step #1. Connect to Your VM via SSH
ssh root@your-public-ip
Replace your-public-ip with the public IP address of your Linode VM.
it is recamended you create a New User profile with sudo Privileges
here's how to add a user if you want to add a user 
adduser [your-username]
usermod -aG sudo [your-username]
Once you've done this, log out of root and back in as your new user.

# Step #1.5 how to update your ubuntu vm
(sudo apt update && sudo apt upgrade -y)
its that simple

# Step #2. Configure UFW Firewall *optinal 

skip to step three if you dont want to do this part

This allows only necessary traffic (SSH, HTTP, and HTTPS).
(sudo ufw allow ssh)
(sudo ufw allow http)
(sudo ufw allow https)
(sudo ufw enable)

# Step 3. Install Elasticsearch and Java
Elasticsearch is a powerful search and analytics engine that requires Java.
(sudo apt install openjdk-17-jre -y)
(Add Elasticsearch GPG Key)
This adds the key needed to verify the authenticity of the Elasticsearch package.
(wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg)

# Step 3.5 Add the Elasticsearch Repository
(echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list)

Install Elasticsearch
to do this First, update the ubuntu packages

(sudo apt update)
then run the install command

(sudo apt install elasticsearch -y)

# Step 4. Configure Elasticsearch to Bind to Your Public IP

This allows you to access Elasticsearch from outside your VM.
First, nano into the yml file

(sudo nano /etc/elasticsearch/elasticsearch.yml)
Find the network.host line and change its value to 0.0.0.0
then save the file by doing Ctrl+X then do enter followed by Y
 
# Step 5. Enable and Start Elasticsearch
use the systemctl command
(sudo systemctl enable elasticsearch.service)
(sudo systemctl start elasticsearch.service)

Verify that Elasticsearch is running and listening on port 9200 by running this command on your local machine:
(curl http://<your-public-ip>:9200)
You should see a JSON response.

# Step 6. Set Up a Web Crawler (Scrapy)
You'll use Scrapy, a powerful Python framework, to build a web crawler.
to get started First, install Python, Pip, and Virtualenv

(sudo apt install python3 python3-pip python3-venv -y)
Next, Create and Activate a Virtual Environment that the scrapy will use

(python3 -m venv ~/scrapy_env)
(source ~/scrapy_env/bin/activate)
Then, Install Scrapy and the Elasticsearch Client, requied the scrape the web for urls

(pip install Scrapy elasticsearch)
Create a Scrapy Project

(scrapy startproject search_crawler)
(cd search_crawler)

# Step 6.5 Create the spider
First, create a file search_crawler/spiders/web_spider.py with the following code. This spider will crawl a website and send the extracted data to Elasticsearch.

import scrapy
from elasticsearch import Elasticsearch
from hashlib import sha256

class WebSpider(scrapy.Spider):
    name = 'web'
    start_urls = ['http://toscrape.com']  # Replace with the website you want to crawl

    es = Elasticsearch([{'host': '<your-public-ip>', 'port': 9200, 'scheme': 'http'}])

    def parse(self, response):
        # Index the current page
        doc_id = sha256(response.url.encode()).hexdigest()
        self.es.index(index='web-index', id=doc_id, document={
            'url': response.url,
            'title': response.css('title::text').get(),
            'text': ' '.join(response.css('p::text').getall())
        })

        # Follow all links on the page
        for link in response.css('a::attr(href)'):
            yield response.follow(link, self.parse)
# Step 7. Run the Spider
Start by doing 
(scrapy crawl web)
This will start crawling the website and sending the data to your Elasticsearch instance.

# Step 8. Build a Simple Front End
This is crusal for displaying the webiste and its information
You'll use Node.js and Express to create a simple web server that serves a basic search interface.

First, install Node.js
(sudo apt install nodejs npm -y)
Then, create a Project Directory and Install Dependencies
(mkdir search_frontend && cd search_frontend)
(npm init -y)
(npm install express elasticsearch)
Create a Server Script that users will use to interact with ui elements

Create a file server.js with the following content below. This script handles search queries and serves the front-end page.

const express = require('express');
const { Client } = require('@elastic/elasticsearch');

const app = express();
const port = 3000;
const client = new Client({ node: 'http://<your-public-ip>:9200' });

app.use(express.static('public'));
app.use(express.json());

app.post('/search', async (req, res) => {
    const { query } = req.body;
    try {
        const { body } = await client.search({
            index: 'web-index',
            body: {
                query: {
                    multi_match: {
                        query,
                        fields: ['title', 'text']
                    }
                }
            }
        });
        res.json(body.hits.hits);
    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'Search failed' });
    }
});

app.listen(port, () => {
    console.log(`Server listening at http://<your-public-ip>:${port}`);
});
Create the Front-End Page that users will use to see the page
Create a public directory and an index.html file inside it.

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Simple Search Engine</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        #results { margin-top: 20px; }
        .result { border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; border-radius: 8px; }
        .result h3 { margin-top: 0; }
    </style>
</head>
<body>
    <h1>My Search Engine</h1>
    <input type="text" id="search-box" placeholder="Search...">
    <button id="search-button">Search</button>
    <div id="results"></div>

    <script>
        document.getElementById('search-button').addEventListener('click', async () => {
            const query = document.getElementById('search-box').value;
            const response = await fetch('/search', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ query })
            });
            const results = await response.json();
            const resultsDiv = document.getElementById('results');
            resultsDiv.innerHTML = '';
            results.forEach(hit => {
                const resultDiv = document.createElement('div');
                resultDiv.className = 'result';
                resultDiv.innerHTML = `<h3><a href="${hit._source.url}" target="_blank">${hit._source.title}</a></h3><p>${hit._source.text.substring(0, 200)}...</p>`;
                resultsDiv.appendChild(resultDiv);
            });
        });
    </script>
</body>
</html>
# Last Step Start the Server
(node server.js)
thats it
Your search engine will now be accessible from your web browser at http://your-public-ip:3000.
