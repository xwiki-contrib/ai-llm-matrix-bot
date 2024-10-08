# AI LLM Matrix Bot

AI LLM Matrix Bot is a chatbot for the [Matrix](https://matrix.org/) chat protocol, designed to interact with [XWiki's LLM](https://extensions.xwiki.org/xwiki/bin/view/Extension/LLM/) application from the Matrix Element chat.

This project is based on [nfinigpt-matrix](https://github.com/h1ddenpr0cess20/infinigpt-matrix/) by h1ddenpr0cess20, with significant modifications.

## Features

- Interacts with XWiki's RAG system for enhanced responses
- Supports setting different AI models per room, configurable through the XWiki interface
- Personalized chat history for each user in each channel
- Ability to assume different personalities or use custom prompts
- Collaborative feature allowing users to interact with each other's chat histories
- Configurable moderation system to filter inappropriate content
- Flexible JWT-based authentication for secure communication with the XWiki server
- Support for E2EE encrypted rooms

## Setup

- Clone the repository:

```
git clone https://github.com/xwiki-contrib/ai-llm-matrix-bot.git
cd ai-llm-matrix-bot
```

- Generate an EdDSA key pair and save the private key as `private.pem` in the project directory.

```
openssl genpkey -algorithm ed25519 -outform PEM -out private.pem
openssl pkey -in private.pem -pubout -outform PEM -out public.pem
```

- Set up a [Matrix account](https://app.element.io/) for your bot.  You'll need the server, username and password.

- Update the configuration file `config.json` according to your needs 
    Fields:
    - `models`: String list of allowed AI models
    - `restrict_to_specified_models`: Boolean value, if true the available models will be restricted to the list set above 
    - `server`: Matrix server URL. (Default value `https://matrix.org`)
    - `xwiki_xwiki_v1_endpoint`: XWiki RAG system endpoint. (default value: `http://localhost:8080/xwiki/rest/wikis/xwiki/aiLLM/v1`)
        - for use with docker on the same network `http://[container-name]:[internal-port]/rest/wikis/xwiki/aiLLM/v1`
    - `matrix_username` and `matrix_password`: Matrix credentials of the bot's user
    - `device_id`: Random string (default value: `4d34ac89fd7346e4aaf4b896763b8f2b`)
    - `admins`: List of Matrix user IDs with admin privileges 
    - `channels`: List of room ids the bot will join if `auto_join_rooms` is set to false 
    - `auto_join_rooms`: Boolean value, if set to true the channels field will be ignored
    - `personality`: Text prompt defining the default personality of the bot
    - `moderation_strategy`: Defines the moderation strategy used (Current supported values: `forbidden_words`)
    - `forbidden_words`: List of words to be filtered in moderation
    - `moderation_enabled`: Boolean value that enables or disables the moderation strategy defined above
    - `default_model`: The default AI model selected
    - `jwt_payload`: Custom claims for the JWT used for XWiki authentication
        - `iss`: the issuer, corresponding to the URL property configured in the authorized applications (can be a string value ex: `matrix-bot`)
        - `aud`: the audience, must contain the URL of the XWiki installation in the form `https://www.example.com/` without path. Both a single string and an array of strings are supported. If the expected URL isn't passed, an error is logged with the expected URL.
            - for use with docker on the same network `http://[container-name]:[internal-port]/`
        - `groups`: a list of groups (as list of strings). Used to set the groups for the bot's user. (Default value: ["MatrixGroup"])
        - `sync_timeout`: sets the timeout value for the Matrix client's sync operation. It determines how long the client will wait for a response from the server during the synchronization process (expressed in miliseconds)
    - `response_temperature`: temperature to be used for the models (value between 0 and 2)
    - `jwt_expiration_hours`: expiration period for the jwt expressed in hours

- Configure the XWiki server to accept JWT tokens from the bot, using https://extensions.xwiki.org/xwiki/bin/view/Extension/LLM/Authenticator/

### Using `docker compose`

- Make sure you have docker installed 
    - Install Docker:
        Follow the official Docker installation guide for your operating system:
        https://docs.docker.com/get-docker/

- Make sure you have Docker Compose installed
    - Docker Compose is included with Docker Desktop for Windows and macOS. For Linux, follow the installation instructions here:
        https://docs.docker.com/compose/install/

- Make sure you are in the cloned repository's folder (see first setup step)

- To verify the docker installation you can use:
```
docker --version
docker compose version
```
These commands should display the installed versions of Docker and Docker Compose.

- Start the container:

```
docker compose up
```

Note: use flag `-d` to run the container in the background and `--build` to rebuild the image.

- View logs:

```
docker logs -f matrix-bot
```

- Stop the container:

```
docker compose down
```

The Docker setup includes:
- A volume for nio_store to persist data
- A volume for the config directory
- Mounted private.pem and public.pem files

### Using `docker run`

Note: Make sure the directories you are mounting into the container are fully-qualified, and aren't relative paths.

To build and run the ai-llm-matrix-bot using Docker directly, follow these steps:

- Build the Docker image:

```
docker build -t ai-llm-matrix-bot -f docker/Dockerfile .
```

- Run the container:

```
docker run -d \
  --name matrix-bot \
  -e CONFIG_PATH=/app/config/config.json \
  -v $(pwd)/nio_store:/app/nio_store \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/private.pem:/app/private.pem \
  -v $(pwd)/public.pem:/app/public.pem \
  --network bridge \
  --log-driver json-file \
  --log-opt max-size=200m \
  --log-opt max-file=10 \
  ai-llm-matrix-bot
```

This command does the following:

- Names the container "matrix-bot"
- Sets the CONFIG_PATH environment variable
- Mounts the necessary volumes for nio_store, config, and key files
- Uses the bridge network
- Configures logging with a max size of 200MB and up to 10 log files


To view the logs:

```
docker logs -f matrix-bot
```

To stop the container:

```
docker stop matrix-bot
```

To remove the container:

```
docker rm matrix-bot
```

### Alternatively using conda

- Install [Conda](https://conda.io/projects/conda/en/latest/user-guide/install/index.html) if you haven't already.

- Make sure you are in the cloned repository's folder (see first setup step)

- Prepare the environment using conda:
```
conda env create -f environment.yml
```

and activate it

```
conda activate matrix-bot
```

- Run the bot:
```
python infinigpt.py
```

## Allow access
Before you can talk to the model you first need allow access to it from the AI LLM application.
A new group defined in the confi.json (by default `MatrixGroup`) shoud be available in your xwiki instance.
Add it to the list of groups that can query the model in the AI LLM application's model configurtion using the `group` field.
Note: on default setups using `XWikiAllGroup` should also work.

## Usage

Users can interact with the bot using the following commands:

- `.ai <message>` or `@botname: <message>`: Send a message to the bot
- `.x <user> <message>`: Interact with another user's chat history
- `.persona <personality>`: Change the bot's personality
- `.custom <prompt>`: Use a custom system prompt
- `.reset`: Reset to the default personality
- `.stock`: Remove personality and reset to standard settings
- `.model`: List available AI models
- `.model <modelname>`: Change the current AI model
- `.model reset`: Reset to the default model
- `.help`: Display the help menu

## Development

This project is actively maintained by the XWiki community. Contributions, bug reports, and feature requests are welcome. Please submit issues and pull requests to the [GitHub repository](https://github.com/xwiki-contrib/ai-llm-matrix-bot).

## License

This project is licensed under [LGPL v2.1](https://www.gnu.org/licenses/lgpl-2.1.en.html).

## Acknowledgements

Special thanks to h1ddenpr0cess20 for the original [infinigpt-irc](https://github.com/h1ddenpr0cess20/infinigpt-irc/) project, which served as the foundation for this Matrix bot.

---

* Project Lead: Ludovic Dubost 
* [Issue Tracker](https://jira.xwiki.org/browse/LLMAI)
* Communication: [Forum](https://forum.xwiki.org/), [Chat](https://dev.xwiki.org/xwiki/bin/view/Community/Chat)
* License: LGPL 2.1
* Translations: N/A
* Sonar Dashboard: N/A
* Continuous Integration Status: N/A
