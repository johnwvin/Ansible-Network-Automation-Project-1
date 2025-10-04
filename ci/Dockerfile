# Step 1: Pull from the temporary registry service container
FROM registry-server:5000/python:3.13-slim

# Step 2: Install from the temporary PyPI service container
RUN pip install --trusted-host pypi-server \
                --index-url http://pypi-server:8080/simple/ \
                ansible-lint==25.9.1 ansible-core==2.19.2