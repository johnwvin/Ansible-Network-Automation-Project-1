# /ci/Dockerfile in your Ansible project repository

# Step 1: Pull the base image FROM your local registry on the agent
FROM localhost:5000/python:3.13-slim

# Step 2: Tell pip to INSTALL from your local PyPI mirror on the agent
# The --trusted-host flag is needed for the non-HTTPS connection.
RUN pip install --trusted-host localhost \
                --index-url http://localhost:8080/simple/ \
                ansible-lint==25.9.1 ansible-core==2.19.2