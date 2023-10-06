# Quality Code Lib
## Objective
This application will allow users to save their Python functions in a code library. The functions are searchable and browsable, publicly available for other users.
The application supports users in documentation and quality control of their functions.

## Features
### Registration
- Users can create new accounts using their email as username

### Saving new functions
- Users can save new functions to the library
- Application will do following checks and provide feedback
    - Linting using pylint
    - Type checking using mypy
    - Security checking using bandit
    - (Optional) Automated documentation using OpenAI API
    - (Optional) Automated generation of unit tests using OpenAI API
    - (Optional) Automated execution of unit tests using AWS Lambda
    - (Optional) Automated keyword classification using OpenAI API
- Users are given possibility to refine their code before saving, based on feedback
- Users can classify their functions using keywords

### Browsing and searching the library
- Users can search and view functions saved by other people
- Users can rate functions and sort by rating

### Library management
- Administrators can manage users and saved functions
- Users can remove their own saved functions

## Database Schema
### Users
| name              | type    | default                                 | constraints     | references | description                             |
|-------------------|---------|-----------------------------------------|-----------------|------------|-----------------------------------------|
| user_id               | UUID    | gen_random_uuid()                       | PRIMARY KEY     |            | unique identifier for all users         |
| username          | TEXT    |                                         | UNIQUE NOT NULL |            | username in plain text                  |
| password          | TEXT    |                                         | NOT NULL        |            | hashed password                         |
| verification_code | TEXT    |                                         | NOT NULL        |            | hashed verification code                |
| admin             | BOOLEAN | FALSE                                   |                 |            | flag is true if user is administrator   |
| verified          | BOOLEAN | FALSE                                   |                 |            | flag is true if user has verified email |
| disabled          | BOOLEAN | FALSE                                   |                 |            | flag is true if user is disabled        |
| created           | INT     | EXTRACT(EPOCH  FROM  CURRENT_TIMESTAMP) |                 |            | timestamp of creation, in unix format   |
| locked            | INT     |                                         |                 |            | timestamp of locking, in unix format    |

### Auth_events
| name       | type    | default                               | constraints | references | description                                                     |
|------------|---------|---------------------------------------|-------------|------------|-----------------------------------------------------------------|
| event_id   | SERIAL  |                                       | PRIMARY KEY |            | unique id for each event                                        |
| user_id        | UUID    |                                       |             | users(user_id) | user related to this event                                      |
| event_time | INT     | EXTRACT(EPOCH FROM CURRENT_TIMESTAMP) |             |            | timestamp of action, in unix format                             |
| event_type | TEXT    |                                       | NOT NULL    |            | either 'login' or 'verification'                                |
| success    | BOOLEAN |                                       | NOT NULL    |            | flag is true if authentication was successful                   |
| remote_ip  | TEXT    |                                       | NOT NULL    |            | remote IP address from which the authentication originated from |
| reason     | TEXT    |                                       |             |            | description of why action failed                                |

### Account_events
| name       | type    | default                               | constraints | references | description                                             |
|------------|---------|---------------------------------------|-------------|------------|---------------------------------------------------------|
| event_id   | SERIAL  |                                       | PRIMARY KEY |            | unique id for each event                                |
| user_id        | UUID    |                                       |             | users(user_id) | user related to this event                              |
| event_time | INT     | EXTRACT(EPOCH FROM CURRENT_TIMESTAMP) |             |            | timestamp of action, in unix format                     |
| event_type | TEXT    |                                       | NOT NULL    |            | always 'create'                                         |
| success    | BOOLEAN |                                       | NOT NULL    |            | flag is true if action was successful                   |
| remote_ip  | TEXT    |                                       | NOT NULL    |            | remote IP address from which the action originated from |
| reason     | TEXT    |                                       |             |            | description of why action failed                        |

### Functions
| name        | type   | default | constraints | references | description                        |
|-------------|--------|---------|-------------|------------|------------------------------------|
| function_id | SERIAL |         | PRIMARY KEY |            | unique id for each function        |
| user_id         | UUID   |         | NOT NULL    | users(user_id) | user who created to this function  |
| name        | TEXT   |         | NOT NULL    |            | name of the function               |
| code        | TEXT   |         | NOT NULL    |            | source code of the function        |
| tests       | TEXT   |         | NOT NULL    |            | source code of the unit tests      |
| usecase     | TEXT   |         | NOT NULL    |            | description of function's use case |
| keywords    | TEXT   |         | NOT NULL    |            | comma separated list of keywords   |

## Status
### Operational functionalities
- Users can create new accounts using their email as username
- Users can save new functions to the library
    - Linting with pylint
    - Automated documentation using OpenAI API
    - Automated generation of unit tests using OpenAI API
    - Automated execution of unit tests using AWS Lambda
    - Automated keyword classification using OpenAI API
- Users can browse (not search) functions saved by other people

### TODO
- Type checking using mypy
- Security checking using bandit
- Search and rating functionality
- User and library management functionality
- Improve user interface and visuals

## Testing
Application is packaged into docker container using Github Action, and uploaded to AWS ECR by the pipeline.
AWS AppRunner is serving the app, which is available at https://codeqa.online

To create an account, you need to use your university email address.

Test account is available:
- username: testuser@villesalmela.fi
- password: password

One test function was saved to the library using this input:
```python
def celsius_to_fahrenheit(celsius):
    fahrenheit = (9/5) * celsius + 32
    return fahrenheit
```
Feel free to try the process with that one, or come up with something else.

### Local Testing
While you can run the web app and database locally, some components don't currently have local options.
1. Install [Docker Engine](https://docs.docker.com/engine/install/)
1. Install [Visual Studio Code](https://code.visualstudio.com)
1. Install [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
1. Setup a PostgreSQL server
    - *ENV: DB_HOST, DB_PORT, DB_PASSWORD, DB_USER, DB_NAME*
1. Create a new [AWS VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html), which will be used to isolate the lambda function
    - Give it one subnet, and no access to other networks
1. Create a new [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html) function
    - *ENV: PYTEST_REGION*
    - Attach the lambda function to the newly created VPC
    - Modify the automatically created [execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html), which the lambda function will assume
        - Set the role to have one policy, [AWSDenyALL](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSDenyAll.html). \
        This will limit any damage that can occur if malicious user manages to escape isolation. 
    - Upload the [code for the lambda function](aws/lambda/lambda_function.py)
    - Create a [resource based policy](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html) that allows invoking this lambda function
    - Create a user, and assign it the policy you just created
        - [Setup an access key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
        - *ENV: PYTEST_KEY_ID, PYTEST_KEY_SECRET*
1. Setup [OpenAI API](https://platform.openai.com/docs/introduction) key
    - *ENV: OPENAI_API_KEY*
1. Setup [Mailjet](https://www.mailjet.com) account
    - *ENV: MJ_APIKEY_PUBLIC, MJ_APIKEY_SECRET*
    - [create a template for transactional email](https://documentation.mailjet.com/hc/en-us/articles/360042952713-Mailjet-s-Email-Editor-for-Transactional-Emails)
        - *ENV: MJ_TEMPLATE_ID*
    - the template must:
        - accept one variable: "VERIFICATION_CODE"
        - include default subject and sender
1. Clone this project to your workstation
1. Change directory to project root
1. Copy the template for environment variables from [this template](.devcontainer/devcontainer.env.example)
    - Paste the template values in a new file, located in `.devcontainer/devcontainer.env`
    - Place secret values in the newly created file
1. In VS Code, press F1
1. Enter command `>dev containers: open folder in container`
    - select the project root folder
    - this will build and start the image, and make the app available at http://localhost:8000