# Contributing a New Feature to Galaxy Core
Instructors: [John Davis](https://github.com/jdavcs), [Ahmed Awan](https://github.com/ahmedhamidawan)

This tutorial walks you through developing an extension to [Galaxy](https://github.com/galaxyproject/galaxy), and how to contribute back to the core project.

### Learning objectives
- Learn about the main principles of Galaxy's code architecture in the context of adding a new feature
- Gain experience working on some of the main parts of the galaxy codebase:
  - Learn how to modify Galaxy's data model and database schema 
  - Learn how to expose a new feature via the API
  - Learn how to implement a new feature in the UI
- Learn about the main libraries used by Galaxy:
  - [SQLAlchemy](https://www.sqlalchemy.org), [Alembic](https://alembic.sqlalchemy.org), [FastAPI](https://fastapi.tiangolo.com), [VueJS](https://vuejs.org/)
- Learn how to use Galaxy's testing frameworks while practicing test-driven development: 
  - Write, execute locally, interpret and debug failed unit tests, api tests, client tests
- Learn about the process of contributing to Galaxy

### Prerequisites
- [Slides: Galaxy Code Architecture](https://training.galaxyproject.org/training-material/topics/dev/tutorials/architecture/slides.html)
- GitHub account, Git, Python, your favorite text editor

## Agenda
1. [Setting up Galaxy locally](#1-setting-up-galaxy-locally)
1. [New feature to implement](#2-new-feature-to-implement)
1. [Updating the data model](#3-updating-the-data-model)
1. [Database schema migrations](#4-database-schema-migrations)
1. [Test Driven Development](#5-test-driven-development)
1. [Implementing the API](#6-implementing-the-api)
1. [Building the UI](#7-building-the-ui)

## 1. Setting up Galaxy locally
To contribute to galaxy, a GitHub account is required. Changes are proposed via a pull request. This allows the project maintainers to review the changes and suggest improvements.

The general steps are as follows:
1. Fork the Galaxy repository
1. Clone your fork
1. Make changes in a new branch
1. Commit your changes, push branch to your fork
1. Open a pull request for this branch in the upstream Galaxy repository

### 1.1 Hands On: Setup your local Galaxy instance

1. Use GitHub UI to fork Galaxy’s repository at galaxyproject/galaxy.
2. Clone your forked repository to a local path, further referred to as GALAXY_ROOT and `cd` into GALAXY_ROOT. Note that we specify the tutorial branch with the `-b` option:

```
$ git clone https://github.com/<your-username>/galaxy GALAXY_ROOT
$ cd GALAXY_ROOT
```

Before we can use Galaxy, we need to create a virtual environment and then activate it:

```
$ python3 -m venv .venv
$ source .venv/bin/activate
```

Once activated, you’ll see the name of the virtual environment prepended to your shell prompt: (.venv)$.

Next, let’s create a new git branch for your edits:

```
$ git checkout -b my-feature
```

Now when you run git branch you’ll see that your new branch is activated:

```
$ git branch
  dev
* my-feature
```

Note: my-feature is just an example; you can call your new branch anything you like.

Finally, as one last step, we need to install the required dependencies and initialize the database. This only applies if you are working on a clean clone and have not started Galaxy. Initializing the database is necessary because you will be making changes to the database schema, which cannot be applied to a database that has not been initialized.

To install the required dependencies and initialize the database, you can simply start Galaxy by executing the following script (might take some time when executing for the first time):

```
$ sh run.sh
```

This will start Galaxy at [http://127.0.0.1:8080](http://127.0.0.1:8080)

## 2. New feature to implement
To setup the proposed extension imagine you’re running a specialized Galaxy server and each of your users only use a few of Galaxy datatypes. You’d like to tailor the UI experience by allowing users of Galaxy to select their favorite datatypes for additional filtering in downstream applications, UI extensions, etc.

Like many extensions to Galaxy, the proposed change requires persistent state. Galaxy stores most persistent state in a relational database. The Python layer that defines Galaxy’s data model is setup by defining SQLAlchemy models.

The proposed extension could be implemented in several different ways on Galaxy’s backend. We will choose one for this example for its simplicity, so we can focus on demonstrating how to modify and extend various layers of Galaxy.

With simplicity in mind, we will implement our proposed extension to Galaxy by adding a single new table to Galaxy’s data model called `user_favorite_datatype`. The concept of a favorite datatype will be represented by a one-to-many relationship from the table that stores Galaxy’s user records to this new table. The datatype itself that will be marked as a favorite will be represented as a filename extension and stored as a text field in this new table. This table will also need to include an integer primary key named `id` to follow the example set by the rest of the Galaxy data model.

## 3. Updating the data model

Galaxy uses a relational database to persist objects and object relationships. Galaxy’s data model represents the object view of this data. To map objects and their relationships onto tables and rows in the database, Galaxy relies on SQLAlchemy, a SQL toolkit and object-relational mapper.

The mapping between objects and the database is defined in `lib/galaxy/model/__init__.py` via declarative mapping, which means that models are defined as Python classes together with database metadata that specifies the database table corresponding to each class.

### 3.1 Add a new class definition

To implement the required changes to add the new model, you need to create a new class in `lib/galaxy/model/__init__.py`

- Add a class definition for your new class. You class should be a subclass of `Base` and `RepresentById`.
- Add a `__tablename__` attribute
- Add a `mapped_column` attribute for the primary key (should be named `id`)
- Add a `mapped_column` attribute to store the datatype
- Add a `mapped_column` attribute that will serve as a reference (foreign key) to the User model
- Use the relationship function to define an association between your new model and the `User` model.

You can add the new class definition right before the definition of the `User` class. Here's what your class definition should look like:

```py
class UserFavoriteDatatype(Base, RepresentById):
    __tablename__ = "user_favorite_datatype"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("galaxy_user.id"))
    datatype: Mapped[str] = mapped_column(String(255))
    user: Mapped["User"] = relationship(back_populates="favorite_datatypes")
```

The defined relationship should be set on both tables, so you need to add a relationship to the `User` class:

```py
favorite_datatypes: Mapped[List["UserFavoriteDatatype"]] = relationship(back_populates="user")
```

### 3.2 Check your code

When a pull request is submitted to the Galaxy repository on github, multiple github actions are triggered that run thousands of tests to verify the correctness of the proposed changes. It is good practice to at least run a liniting check against your code locally before opening a pull request.

First, we must install ruff:

```
$ pip install -r lib/galaxy/dependencies/pinned-lint-requirements.txt

```

Next, we can check our code:

```
$ ruff check .
All checks passed!
```

### 3.3 Commit your changes

It is good practice to split up your contribution into logical parts: that makes it easier to review the changes, as well as debug the code when necessary, now or in the future. Now is a good time to commit your changes.

First, we review what we're about to commit:

```
$ git status
$ git diff
```

Now let's stage your changes:

```
$ git add --all
```

And, finally, commit, giving your commit an appropriate description:

```
$ git commit -m "Add UserFavoriteDatatype model"
```

And we're done with this step!

Here's what your commit will look like for this step: [3d10657b45c798826abeea896ce82e402f40a688](https://github.com/jdavcs/galaxy/commit/3d10657b45c798826abeea896ce82e402f40a688)

## 4. Database schema migrations

There is one last database issue to consider before moving on to the API. Each successive release of Galaxy requires recipes for how to migrate old database schemas to updated ones. These recipes are called revisions, or migrations, and are implemented using Alembic.

Database schema changes for Galaxy's data model are defined here: `lib/galaxy/model/migrations/alembic/versions_gxy`

We encourage you to read [Galaxy’s documentation on migrations](https://docs.galaxyproject.org/en/master/admin/db_migration.html), as well as relevant [Alembic documentation](https://alembic.sqlalchemy.org/en/latest/tutorial.html#create-a-migration-script).

For this tutorial, you’ll need to do the following:
1. Create a revision template.
1. Edit the revision template, filling in the body of the upgrade and downgrade functions.
1. Run the migration.

### 4.1 Create migration template

First, let's create a blank migration:

```
$ sh scripts/db_dev.sh revision -m "Add user_favorite_datatype table"
```

This command will generate a blank migration template and save it in the `versions_gxy` directory (as noted above). The code in the template will look like this (your hashes will be different):

```py
"""Add user_favorite_datatype table

Revision ID: 6b8c34a345f5
Revises: c716ee82337b
Create Date: 2025-06-22 17:53:20.645121

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '6b8c34a345f5'
down_revision = 'c716ee82337b'
branch_labels = None
depends_on = None


def upgrade():
    pass


def downgrade():
    pass
```

### 4.2 Fill in the migration template

To fill in the revision template, you need to populate the body of the upgrade and downgrade functions. The upgrade function is executed during a schema upgrade, so it should create your table. The downgrade function is executed during a schema downgrade, so it should drop your table. Note that although the table creation command looks similar to the one we used to define the model, it is not the same.

Your final migration code will look like this (note the required import statement):

```py
import sqlalchemy as sa

from galaxy.model.migrations.util import (
    create_table,
    drop_table,
)

# revision identifiers, used by Alembic.
revision = "11a0b86fe248"
down_revision = "c716ee82337b"
branch_labels = None
depends_on = None

TABLE_NAME = "user_favorite_datatype"


def upgrade():
    create_table(
        TABLE_NAME,
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("user_id", sa.Integer, sa.ForeignKey("galaxy_user.id")),
        sa.Column("datatype", sa.String(255)),
    )


def downgrade():
    drop_table(TABLE_NAME)
```

### 4.3 Run the migration and verify the result

Now we are ready to run the migration:

```
$ sh manage_db.sh upgrade
Activating virtualenv at .venv
INFO:alembic.runtime.migration:Context impl SQLiteImpl.
INFO:alembic.runtime.migration:Will assume non-transactional DDL.
INFO:alembic.runtime.migration:Running upgrade c716ee82337b -> 6b8c34a345f5, Add user_favorite_datatype table
DEBUG:alembic.runtime.migration:update c716ee82337b to 6b8c34a345f5
INFO:alembic.runtime.migration:Context impl SQLiteImpl.
INFO:alembic.runtime.migration:Will assume non-transactional DDL.
```

Now let's examine the database. You database is specified in the `database_connection` property in the main Galaxy config file typically located in `config/galaxy.yml`. We don't have a config file, so Galaxy is using the default: `config/galaxy.yml.sample`. If you check that file, you'll see that the default database is located at `database/universe.sqlte`. Let's examine the database: use the `sqlite3` command to launch the database CLI, then use the `.schema` command to view the definition of the new table that has been added:

```
$ sqlite3 database/universe.sqlite
SQLite version 3.34.1 2021-01-20 14:10:07
Enter ".help" for usage hints.
sqlite> .schema user_favorite_datatype
CREATE TABLE user_favorite_datatype (
	id INTEGER NOT NULL,
	user_id INTEGER,
	datatype VARCHAR(255),
	CONSTRAINT user_favorite_datatype_pkey PRIMARY KEY (id),
	CONSTRAINT user_favorite_datatype_user_id_fkey FOREIGN KEY(user_id) REFERENCES galaxy_user (id)
);
```

You may also execute the `manage_db.sh dv` command which will display the database version: one of the hashes will be the hash of the new migration:

```
$ sh manage_db.sh dv
Activating virtualenv at .venv
INFO:alembic.runtime.migration:Context impl SQLiteImpl.
INFO:alembic.runtime.migration:Will assume non-transactional DDL.
d4a650f47a3c (head)
6b8c34a345f5 (head)
```

### 4.4 Check your code and commit

Check your code and fix any errors if needed:

```
$ ruff check .
```

Commit the migration:

```
$ git add --all
$ git commit -m "Add migration"
```

Here's what your commit will look like for this step: [7ef8f88395c0afc00181b14c92d35ae06196463d](https://github.com/jdavcs/galaxy/commit/7ef8f88395c0afc00181b14c92d35ae06196463d)

## 5. Test Driven Development

With the data model and database migration in place, we need to start adding the rest of the Python plumbing required to implement this feature. We will do this with a test-driven approach and start by implementing an API test that exercises operations we would like to have available for favorite extensions.

Note: API tests are only one example of tests we may want to write for this feature. We recommend studying Galaxy's documentation on [how to write tests](https://docs.galaxyproject.org/en/master/dev/writing_tests.html), as well as [this GTN tutorial](https://training.galaxyproject.org/training-material/topics/dev/tutorials/writing_tests/tutorial.html).

### 5.1 Design the test

We will add a test case for user favorite datatypes in `lib/galaxy_test/api/test_users.py` which is a module that contains tests for other user API endpoints. Various user-centered operations have endpoints under `api/user/<user_id>` and `api/user/current` is sometimes substituable as the current user. We will keep things simple and only implement this functionality for the current user.

Our test will test the following three simple API endpoints:

| method | endpoint | description |
| --- | --- | --- |
| GET | `<galaxy_root_url>/api/users/current/favorite_datatypes` | This should return a list of favorited datatypes for the current user |
| POST | `<galaxy_root_url>/api/users/current/favorite_datatypes/<datatype>` | This should mark a datatype as a favorite for the current user |
| DELETE | `<galaxy_root_url>/api/users/current/favorite_datatypes/<datatype>` | This should unmark a datatype as a favorite for the current user |

Our test will do the following:

1. Verify a user initially has no favorite datatypes.
1. Verify that a POST to `<galaxy_root_url>/api/users/current/favorite_datatypes/fasta` returns a 200 status code indicating success.
1. Verify that after this POST the list of user favorited datatypes contains fasta and is of size 1.
1. Verify that a DELETE to `<galaxy_root_url>/api/users/current/favorite_datatypes/fasta` succeeds.
1. Verify that after this DELETE the favorite datatypes list is again empty.

Here's one way to implement such a test (your version might be different):

```py
    def test_favorite_datatypes(self):
        # verify no datatypes
        index_response = self._get("users/current/favorite_datatypes")
        index_response.raise_for_status()
        index = index_response.json()
        assert isinstance(index, list)
        assert len(index) == 0

        # add datatypesAdd commentMore actions
        create_response = self._post("users/current/favorite_datatypes/fasta")
        create_response.raise_for_status()
        create_response = self._post("users/current/favorite_datatypes/fastq")
        create_response.raise_for_status()

        # verify added
        index_response = self._get("users/current/favorite_datatypes")
        index_response.raise_for_status()
        index = index_response.json()
        assert isinstance(index, list)
        assert len(index) == 2
        assert "fasta" in index
        assert "fastq" in index

        # delete datatype
        delete_response = self._delete("users/current/favorite_datatypes/fasta")
        delete_response.raise_for_status()

        # verify deleted
        index_response = self._get("users/current/favorite_datatypes")
        index_response.raise_for_status()
        index = index_response.json()
        assert isinstance(index, list)
        assert len(index) == 1
        assert "fastq" in index
```

### 5.2 Run the test

To run the test, we need to install pytest:

```
$ pip install pytest
```

Then we can run the new test we have just added:

```
pytest lib/galaxy_test/api/test_users.py::TestUsersApi::test_favorite_datatypes
```

(Note: if you see an error referring to `psycopg*` modules, run `pip uninstall pytest-postgresql`)

Of course, the test will fail, but that's expected: we haven't implemented the API yet. 

Before we move on to the next step, let's check our code and commit.

### 5.3 Check your code and commit

Check your code and fix any errors if needed:

```
$ ruff check .
```

Commit the new test:

```
$ git add --all
$ git commit -m "Add API test"
```

Here's what your commit will look like for this step: [1c0a4aaadc407ee5030c1bef80124a262edc480b](https://github.com/jdavcs/galaxy/commit/1c0a4aaadc407ee5030c1bef80124a262edc480b)

## 6. Implementing the API

We'll add our API implementation as a new module at `lib/galaxy/webapps/galaxy/api/user_favorite_datatypes.py`.

### 6.1 Add the code that handles the API endpoints

We have three endpoints to implement. Galaxy relies on [FastAPI](https://fastapi.tiangolo.com/) to handle the low-level complexities of sending and receiving data via HTTP. We encourage you to study its excellent documentation, as well as the many examples of its usage in the context of Galaxy located at `lib/galaxy/webapps/galaxy/api/`.

The overall approach is to define a class with three methods, one for each endpoint. The methods describe the endpoints and delegate the execution of the actual tasks to a user manager.

Here's the code you need for this step:

```py
from typing import List

from fastapi import Path

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.users import UserManager
from . import (
    depends,
    DependsOnTrans,
    Router,
)

router = Router(tags=["user_favorites"])

DatatypePath: str = Path(
    ...,  # Mark this Path parameter as required
    title="Datatype",
    description="Target file extension for target operation.",
)


@router.cbv
class FastAPIUserFavoriteDatatypes:
    user_manager: UserManager = depends(UserManager)

    @router.get(
        "/api/users/current/favorite_datatypes",
        summary="List user favorite datatypes",
        response_description="List of datatypes",
    )
    def index(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> List[str]:
        """Gets the list of user's favorite datatypes."""
        return self.user_manager.get_favorite_datatypes(trans.user)

    @router.post(
        "/api/users/current/favorite_datatypes/{datatype}",
        summary="Mark a datatype as the current user's favorite.",
    )
    def create(
        self,
        datatype: str = DatatypePath,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> str:
        self.user_manager.add_favorite_datatype(trans.user, datatype)
        return datatype

    @router.delete(
        "/api/users/current/favorite_datatypes/{datatype}",
        summary="Unmark a datatype as the current user's favorite.",
    )
    def delete(
        self,
        datatype: str = DatatypePath,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> None:
        self.user_manager.delete_favorite_datatype(trans.user, datatype)
```

Before we move on to the next step, let's check our code and commit.

### 6.2 Check your code and commit

Check your code and fix any errors if needed:

```
$ ruff check .
```

Commit the new test:

```
$ git add --all
$ git commit -m "Implement API"
```

Here's what your commit will look like for this step: [199f796b346a324c047984e86c0fa921885a0fa9](https://github.com/jdavcs/galaxy/commit/199f796b346a324c047984e86c0fa921885a0fa9)

### 6.3 Add the code that handles the actual tasks

The code that handles the tasks described by the endpoints should be located in the user manager at
`lib/galaxy/managers/users.py`.

The user manager is where the actual task is implemented. Each method interacts with the database via Galaxy's data access layer, implemented with the help of SQLAlchemy. The methods are surprisingly simple: retrieving a user's favorite datatypes is a one-liner: the relationship we added to the User model at the beginning of this tutorial handles all the complexity for us. The add and delete methods are almost as simple. To add an item we instantiate a new model, add it to the session (which is responsible for keeping the state of the data objects in Galaxy in sync with the rows in the database that reflect their state). To delete an item we create a delete statement that knows how to generate the required SQL, and let the session execute it. In all three cases, the complexity of transactional communication with the database is handled by SQLAlchemy.

Here's the code you need for this step. Note the additional import statement. The method definitions should be added to the `UserManager` class; for example, you could add them
right after the `__init__` method, but that's not a requirement.


```py
from sqlalchemy import delete  # You need this import!

# The following code should be added to the UserManager class:

    def get_favorite_datatypes(self, user):
        return [fd.datatype for fd in user.favorite_datatypes]

    def add_favorite_datatype(self, user, datatype):
        fd = model.UserFavoriteDatatype(datatype=datatype)
        user.favorite_datatypes.append(fd)
        self.session().add(user)
        self.session().commit()

    def delete_favorite_datatype(self, user, datatype):
        stmt = delete(model.UserFavoriteDatatype).where(model.UserFavoriteDatatype.datatype == datatype)
        self.session().execute(stmt)
        self.session().commit()
```

### 6.4 Run the test

Now we can finally rerun our test!

```
pytest lib/galaxy_test/api/test_users.py::TestUsersApi::test_favorite_datatypes
```

Your test should pass. If not: scroll up the beginning of the pytest output and read the error message carefully - it should point you to the source of the error.

### 6.5 Check your code and commit

Check your code and fix any errors if needed:

```
$ ruff check .
```

Commit the new test:

```
$ git add --all
$ git commit -m "Add manager methods"
```

Here's what your commit will look like for this step: [a3dcd89d7ea195fb8edcbd524898dc61b46ac16d](https://github.com/jdavcs/galaxy/commit/a3dcd89d7ea195fb8edcbd524898dc61b46ac16d)

## 7. Building the UI

Once the API test is done, it is time to build a user interface for this addition to Galaxy. Let’s get some of the plumbing out of the way right away. We’d like to have a URL for viewing the current user’s favorite datatypes in the UI. Let's use this one: `/user_favorite_datatypes` (i.e., in the context of locally running galaxy, it might be `http://127.0.0.1:8080/user_favorite_datatypes`.

The Galaxy front-end, or client, is made up of many components created in Vue; most of them use Vue's Composition API and are implemented in Typescript. To learn more we encourage you to study [Vue's documentation](https://vuejs.org/guide/introduction.html). Of course, there are numerous online resources on [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript) and [TypeScript](https://www.typescriptlang.org/).

To enable our new route, at the minimum we need to do the following:
1. Create a minimal Vue component (we'll worry about implementing the actual functionality in the next step)
1. Register the new route on the backend
1. Register the new route on the client.

### 7.1 Create minimal Vue component

We create the component at `client/src/components/User/UserFavoriteDatatypes.vue`. We start with an empty script block and a minimal template:

```vue
<script setup lang="ts">

</script>

<template>
	<div>
        <h1>Favorite User Datatypes</h1>
    	<p>to do</p>
	</div>
</template>
```

### 7.2 Register the new route on the backend

We register our route on the backend by adding the following line to `lib/galaxy/webapps/galaxy/buildapp.py`:

```py
webapp.add_client_route("/user_favorite_datatypes")
```

Add this line next to similar lines that register other client routes.

### 7.3 Register the new route on the client.

To register our route on the client, we modify `client/src/entry/analysis/router.js`:

Import the new component:

```javascript
import UserFavoriteDatatypes from "@/components/User/UserFavoriteDatatypes.vue";
```

Add the route to `Analysis` routes in the `getRouter` function:

```javascript
{
    path: "user_favorite_datatypes",
    component: UserFavoriteDatatypes,
}
```

### 7.4 Run the development server

When we work on the client, it helpes to have a development server running that will rebuild and reload the code it serves every time you save. For this to work, we need to run the client development server and run Galaxy at the same time:

The following command starts the client development server at port 8081:

```
$ make client-dev-server
```

Now run Galaxy (on port 8080), but we can skip the client build this time, because we will be running the dev server proxied against 8080:

```
$ GALAXY_SKIP_CLIENT_BUILD=1 sh run.sh
```

OR, we can run the equivalent Make command:

```
$ make skip-client
```

**NOTE: You must login before proceeding.** Since this is a new Galaxy, you'll need to register (follow the normal registration steps, except you don't need a secure password since it's your local development copy.

Now you can view your empty page at [http://127.0.0.1:8081/user_favorite_datatypes](http://127.0.0.1:8081/user_favorite_datatypes).

With the dev server running, whenever you make a change to the client code (this *does not* affect changes to the backend code), you will be able to see those changes live (it takes a few seconds to rebuild) at [http://127.0.0.1:8081](http://127.0.0.1:8081). 

*Note: the port is 8081; your regular Galaxy is running at port 8080.*

### 7.5 Commit your code

```
$ git add --all
$ git commit -m "Add blank page and route"
```

Here's what your commit will look like for this step: [edbf7790465d6296142ddfe80be328b78d676a79](https://github.com/jdavcs/galaxy/commit/edbf7790465d6296142ddfe80be328b78d676a79)

### 7.6 Implement client-side functionality

Now we must implement the UI functionality that enables a user to select (and deselect) datatypes. For that, we will utilize four API endpoints: the three that we have added, and an existing one that returns all Galaxy datatypes: `api/datatypes`. Assuming your Galaxy is running, you can view the data you'll need here:
- [http://127.0.0.1:8080/api/datatypes](http://127.0.0.1:8080/api/datatypes): all datatypes
- [http://127.0.0.1:8080/api/users/current/favorite_datatypes](http://127.0.0.1:8080/api/users/current/favorite_datatypes): favorite datatypes (empty so far)

Here's one way to design a simple UI for the required functionality:
- display a list of all datatypes
- if a datatype has not been marked as favorite, display a "+" link
- if a datatype has been marked as a favorit, display a "x" link
- clicking the "+" link marks the datatype as a favorite
- clicking the "x" link unmarks the datatype as a favorite

The following code implements this UI:

```vue
<script setup lang="ts">
import axios from "axios";
import { onMounted, ref } from "vue";

const datatypes = ref<any[]>([]);
const favoriteDatatypes = ref<any[]>([]);

onMounted(async () => {
    await loadDatatypes();
    await loadFavoriteDatatypes();
});

async function loadDatatypes() {
	const path = 'api/datatypes';
	try {
    	const response = await axios.get(path);
    	datatypes.value = response.data;
    } catch (error) {
    	console.error(error);
	}
}

async function loadFavoriteDatatypes() {
	const path = 'api/users/current/favorite_datatypes';
	try {
    	const response = await axios.get(path);
    	favoriteDatatypes.value = response.data.sort();
    } catch (error) {
    	console.error(error);
	}
}

async function mark(datatype) {
    const path = `api/users/current/favorite_datatypes/${datatype}`;
	try {
    	await axios.post(path);
        await loadFavoriteDatatypes();
    } catch (error) {
    	console.error(error);
	}
}

async function unmark(datatype) {
    const path = `api/users/current/favorite_datatypes/${datatype}`;
	try {
    	await axios.delete(path);
        await loadFavoriteDatatypes();
    } catch (error) {
    	console.error(error);
	}
}

</script>

<template>
	<div>
        <h1>Favorite Datatypes</h1>

        <ul>
        	<li v-for="dt in datatypes" :key="dt">
                {{ dt }}
                <a @click="unmark(dt)" href="#" v-if="favoriteDatatypes.indexOf(dt) >= 0" class="red">X</a>
                <a @click="mark(dt)" href="#" v-else class="green">+</a>
            </li>

    	</ul>

	</div>
</template>

<style>
.red {
    color: red;
}
.green {
    color: green;
}
</style>
```



If you'd like to see the same changes show up on the original proxy (on port 8080), run the `run.sh` command to ensure that the client code is rebuilt (since we won't be skipping the client build this time):

```
$ sh run.sh
```

Now you can view your new page at the default port 8080: [http://127.0.0.1:8080/user_favorite_datatypes](http://127.0.0.1:8080/user_favorite_datatypes).

![image](https://github.com/user-attachments/assets/f99e7c3a-464d-4e41-9682-a3fcf14161ae)

Try it out! Try to mark and unmark datatypes.

(Note: if nothing happens, and your browser console output shows an error, check whether you are logged in!)

### 7.7 Commit your code

```
$ git add --all
$ git commit -m "Display all datatypes"
```

Here's what your commit will look like for this step: [43f2b74deb1980b7f7933e5c1ccd3211f511dbf0](https://github.com/jdavcs/galaxy/commit/43f2b74deb1980b7f7933e5c1ccd3211f511dbf0)

## Congratulations! You've completed the tutorial!
