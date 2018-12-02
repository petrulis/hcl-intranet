# Sample Intranet

## Architecture

![Diagram](/docs/architecture.png)

## Setup

openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout cert.key -out cert.pem -config req.cnf -sha256
aws acm import-certificate --certificate file://cert.pem --private-key file://cert.key --region eu-west-1