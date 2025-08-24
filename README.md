# Linode Seach Engine
A project made to help user make a search engine on linode
# The Guide
Step 1: Set Up and Secure Your Linode VM
This guide requies you to have a Linode account and to have a created Ubuntu 24.04 LTS VM.
# Step #1. Connect to Your VM via SSH
ssh root@<your-public-ip>
Replace <your-public-ip> with the public IP address of your Linode VM.
it is recamended you create a New User profile with sudo Privileges

here's how to add a user if you want to add a user 
adduser <your-username>
usermod -aG sudo <your-username>
Once you've done this, log out of root and back in as your new user.

heres how to update Your ubuntu system
sudo apt update && sudo apt upgrade -y

Configure UFW Firewall *optinal 
This allows only necessary traffic (SSH, HTTP, and HTTPS).
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable

Install Elasticsearch and Java
Elasticsearch is a powerful search and analytics engine that requires Java.

sudo apt install openjdk-17-jre -y
Add Elasticsearch GPG Key

This adds the key needed to verify the authenticity of the Elasticsearch package.

Bash

wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
Add the Elasticsearch Repository

This adds the official repository to your system's sources.

Bash

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
Install Elasticsearch

Bash

sudo apt update
sudo apt install elasticsearch -y
Configure Elasticsearch to Bind to Your Public IP

This allows you to access Elasticsearch from outside your VM.

Bash

sudo nano /etc/elasticsearch/elasticsearch.yml
Find the network.host line and change its value to 0.0.0.0.

YAML

# ---------------------------------- Network -----------------------------------
#
# Set the address and port where this node will be reachable.
#
network.host: 0.0.0.0
Save the file and exit the editor.

Enable and Start Elasticsearch

Bash

sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
Verify that Elasticsearch is running and listening on port 9200 by running this command on your local machine:

Bash

curl http://<your-public-ip>:9200
You should see a JSON response.

Step 3: Set Up a Web Crawler (Scrapy)
You'll use Scrapy, a powerful Python framework, to build a web crawler.

Install Python, Pip, and Virtualenv

Bash

sudo apt install python3 python3-pip python3-venv -y
Create and Activate a Virtual Environment

Bash

python3 -m venv ~/scrapy_env
source ~/scrapy_env/bin/activate
Install Scrapy and the Elasticsearch Client

Bash

pip install Scrapy elasticsearch
Create a Scrapy Project

Bash

scrapy startproject search_crawler
cd search_crawler
Create a Spider

Create a file search_crawler/spiders/web_spider.py with the following code. This spider will crawl a website and send the extracted data to Elasticsearch.

Python

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
Run the Spider

Bash

scrapy crawl web
This will start crawling the website and sending the data to your Elasticsearch instance.

Step 4: Build a Simple Front End
You'll use Node.js and Express to create a simple web server that serves a basic search interface.

Install Node.js

Bash

sudo apt install nodejs npm -y
Create a Project Directory and Install Dependencies

Bash

mkdir search_frontend && cd search_frontend
npm init -y
npm install express elasticsearch
Create a Server Script

Create a file server.js with the following content. This script handles search queries and serves the front-end page.

JavaScript

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
Create the Front-End Page

Create a public directory and an index.html file inside it.

HTML

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
Start the Server

Bash

node server.js
Your search engine will now be accessible from your web browser at http://<your-public-ip>:3000.
