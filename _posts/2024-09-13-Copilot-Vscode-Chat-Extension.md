---
layout: post
title: Xebia Innovation Day - Creating a GitHub Copilot VSCode Chat Extension
date: 2024-09-13 20:00 -0400
description: >
  Today was Innovation Day at Xebia, and for my team's project we decided to try creating a VSCode Chat extension for GitHub Copilot.  The extension would allow you to utilize Copilot's AI capabilities to help write SQL queries without having to directly tell Copilot about the database schema - something that Copilot currently struggles with.  We learned a lot of interesting things through this project, and I'm excited to share them with you!
#img: github-copilot-custom-instructions/hero.jpeg
#fig-caption: An AI assistant reading some custom instructions, dressed as a pirate who loves oranges.
tags: [GitHub, Copilot, Coding, Development, AI, VSCode, Innovation Day]
---

Before we begin, I just want to say thank you to Xebia for encouraging all of us to be innovative, explore, learn, and share our knowledge with others.  I am truly blessed to work for such an awesome employer.  Today was Innovation Day at Xebia, which is one of my favorite parts of being a consultant here.  On Innovation Day, we shut down our regular client work and instead team up to try to create something new and interesting (and maybe _innovative_).  There are two main rules - you have to learn something new, and you cannot work on anything client-related.  At the end of the day, we all come together to show off what we did and what we learned, and it's always a lot of fun.

For my project today, I decided to work with a small team to create a VSCode Chat extension for GitHub Copilot.  Inspired by [a demo I saw at Build](https://youtu.be/OdW2r3raAHI?si=3FOq9JGGytpGLYqr), we decided to go one step further than just writing a simple extension - we wanted to create an extension that would allow you to utilize Copilot's AI capabilities to help write SQL queries without having to directly tell Copilot about the database schema.  This is something that Copilot currently struggles with, and we thought it would be a fun challenge.  And what a challenge it turned out to be.

## The Idea

GitHub Copilot only takes it's context from code files.  In VSCode, there's an extension (the [mssql extension](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql)) that allows you to connect to a database and run queries. Unfortunately, Copilot doesn't know how to use this extension, so the only way to get Copilot to help with writing queries is to have the schema in workspace files, or to pass the schema as part of your prompt.  Once Copilot knows the Schema it's working with, it's been very good at helping write queries, but we wanted to see if we could make it even easier by having Copilot read the schema from the database directly.  Sounds simple enough, right?  Welllllll....

## Step 1 - Scaffold The VSCode Chat Extension

As it turns out, this was the simplest part of the day.  VSCode chat extensions are surprisingly easy to write.  In fact, I walked into today having already demo'd a chat extension that I wrote for one of our Xebia Knowledge Exchange sessions.  This extension was a simple chatbot that answered questions in the voice of various Star Wars characters (the code is available in the [molson504x/copilot-custom-extension](https://github.com/molson504x/copilot-custom-extension) repo on my GitHub).  There's a bit of boilerplate code, so let's dive in.

### Define the extension

VSCode Extensions are written in TypeScript, and contain a `package.json` file which defines some information about the extension.  Chat Extensions in VSCode are no different, but they contain one extra bit of information - the `contributes.chatParticipants` section.  This is where you define information about Chat Participants, namely the ID, Name, Description, and Commands which a user would use to interface with a participant.  It looks something like this:

``` json
{
    "contributes": {
        "chatParticipants": [
            {
                "id": "xebia.sql-helper",
                "name": "Sql-Helper",
                "description": "Helps write SQL queries based on the open database connection in the mssql extension",
                "isSticky": true
            }
        ]
    },
}
```

This information also helps VSCode know what to display when you run the `/help` command in the chat window, so it's important that this information provides enough information to the user to know how to interact with the participant.  Think of this as your documentation.

### Define The Chat Participant Handler

Chat commands need to be handled by a handler.  Chat handlers define how the extension should handle the user's prompts and commands, and also is responsible for streaming progress indications and output indications to the chat window.  This is a boilerplate implementation of our chat handler, before we added any other logic to it:

``` typescript
export async function sqlHelperHandler(request: vscode.ChatRequest, context: vscode.ChatContext, stream: vscode.ChatResponseStream, token: vscode.CancellationToken): Promise<IChatResult> {
    stream.progress('Getting you a SQL query to run...');
    try {
        const [model] = await vscode.lm.selectChatModels(MODEL_SELECTOR);
        if (model) {
            // Do something useful here, this is just boilerplate...
            stream.markdown(`User's Prompt: ${request.prompt}`);
        }
    }
    catch (err) {
        handleError(logger, err, stream);
    }

    logger.logUsage('request', {kind: 'sql-helper', prompt: request.prompt});
    return {metadata: {command: request.command ?? 'sql-helper', prompt: request.prompt}};
};
```

This handler is pretty simple - it just logs the user's prompt to the chat window.  It also selects which model we want to use for servicing our prompts later on, and `MODEL_SELECTOR` is a constant which points to the GPT-4o model.  We'll add more to it later, but this is a good starting point.  I also had boilerplate for error handling and logging, but I won't bore you with that.  It just logs debug information to the console.

### Register the Chat Participant

The last step in creating a chat participant is to register it with the VSCode API.  This is done in the `activate` function of your extension, and looks something like this:

``` typescript
export function activate(context: vscode.ExtensionContext) {
    const sqlHelper = vscode.chat.createChatParticipant(SQL_HELPER_PARTICIPANT_ID, sqlHelperHandler);
}
```

In our case, `SQL_HELPER_PARTICIPANT_ID` is a constant that we defined which matches the ID from our `package.json` file.  This is how VSCode knows which handler to call when a user interacts with the chat participant.  And that's it, right?  Well, kinda.  Here's where we had to start getting creative.

## Step 2 - Connecting to the Database (aka - The Hard Part)

The next step was to call the mssql extension to connect to the database and get the schema.  This turned out to be a bit harder than we thought, and since we were running low on time (and this was a proof-of-concept) we decided to cheat a little and instead just create a command to set the Connection String and bypass the mssql extension entirely (remember that `/connect` command in our chat participant declaration?).  We'll go back and fix that later (maybe) but for now, we just wanted to get something working.  Our goal was to show off Copilot, not the mssql extension.

### Our Database Helper

We created a helper class with one goal in mind - interface with the mssql extension.  This was the hardest part of the day, but definitely the most rewarding to see it working.  We used a little help from Copilot to figure out how to do this, and also had to figure out how to interface with the VSCode Language Service (and how they implemented this service for SQL).  At the end, this turned out to be a simple few lines of code that took close to 6 hours to figure out.

Let's break down what did.

First, was getting a reference to the extension activating it if it hadn't already been activated.  In theory, we shouldn't have had to do this but it's good to validate things.  We also went ahead and got a reference to the mssql extension's exports, which we used as our "API" to interact with the extension.

``` typescript
const mssqlExtension = vscode.extensions.getExtension('ms-mssql.mssql');
if (!mssqlExtension.isActive) {
    await mssqlExtension.activate();
}
const mssqlApi = mssqlExtension.exports;
```

Next, we had to connect to the database.  This was a bit tricky to understand since it required two steps - retrieve a stored connection profile URI, and then use that to connect to the database.  It's two lines of code, but it took us a long time to understand - mostly because of the URI that gets returned from the `connect()` call, which looks weird (something like `vscode-mssql-adhoc://query0`).  This threw us through a loop since we didn't understand what this URI was used for, but eventually we figured it out.

``` typescript
const connectionProfile = await mssqlApi.promptForConnection();
const connection = await mssqlApi.connect(connectionProfile.connectionUri);
```

Our query that we used to get the table schema was pretty standard:

``` sql
SELECT CONCAT(tables.TABLE_SCHEMA, '.', tables.TABLE_NAME) AS TABLE_NAME, columns.COLUMN_NAME, columns.DATA_TYPE
FROM INFORMATION_SCHEMA.TABLES as tables
INNER JOIN INFORMATION_SCHEMA.COLUMNS as columns ON tables.TABLE_NAME = columns.TABLE_NAME
WHERE tables.TABLE_TYPE = 'BASE TABLE'
```

And we then had to figure out how to send this query through the mssql extension.  We started by digging through [the mssql extension code](https://github.com/microsoft/vscode-mssql), which is where we found information on what `sendRequest()` does - it sends a command and parameters to a `SqlToolsServiceClient`, which is just an extension of a `LanguageClient`.  If it's a _client_ that means there's a _server_ somewhere, right?  This is where things got tricky, and we eventually found the documentation on the [VSCode Language Server](https://github.com/microsoft/vscode-languageserver-node), and then from there wound up finding the repository which contains the implementation of this service for SQL Server ([microsoft/sqltoolsservice](https://github.com/microsoft/sqltoolsservice)).  From there, we found information on the JSON-RPC Protocol used to communicate with this service, and eventually found the right RPC command to pass in along with its parameters.  This was a bit of a rabbit hole, but we eventually got it working.

``` typescript
const result = await mssqlApi.sendRequest('query/simpleexecute', { ownerUri: connectionUri, queryString: tableColumnQuery });
```

No, really, that's it.  One line of code that took us HOURS to figure out.  From there, we flattened the 2D array representation of the returned rows into our desired JSON object and then returned it to our chat handler.

We want to go back eventually and clean this implementation up so that it doesn't prompt the user every time they use this chat handler, but this is a proof-of-concept and it was a miracle that we got it working at all.  We were pretty happy with this result.

### Passing the Schema to Copilot

Once we had the helper class written, we could then get the schema and pass it into Copilot as part of our prompt.  This was also a bit tricky, and required us to make a couple of assumptions for this POC:

1. This would be a small enough schema to avoid hitting the token limit of the model.
2. We only cared about the table columns, not indexes, FK's, etc...
3. We would go back later to pre-flight the schema and only pass the table schema for tables that were relevant to the user's prompt.

To do this, we dynamically rendered a prompt (a series of messages) to pass into Copilot by using a `PromptElement` implementation along with the vscode `renderPrompt()` function.  A PromptElement is a way to define a series of messages which should be rendered and sent to Copilot as the prompt.  For example, this is the PromptElement we used to render the message containing the schema definition:

``` typescript
class TableSchemaPrompt extends PromptElement<PromptProps, void> {
    render(state: void, sizing: PromptSizing) {
        return (
            <>
                <UserMessage>
                    ## Table Schema (Database Schema)
                    
                    This is a JSON representation of the table schema for the database being asked about.
                    The schema is described as follows:

                    Each object in the array represents a table in the database.  The key is the name of the table.
                    The value is an array of strings, where each string represents a column in the table along with its data type.

                    ``` JSON
                    {JSON.stringify(this.props.tableSchema)}
                    ```
                </UserMessage>
            </>
        );
    }
}
```

Additionally, a PromptElement can contain other PromptElements, so we could have a series of messages that would be sent to Copilot as part of the prompt.  This is what we actually rendered as our prompt:

``` typescript
export class SqlHelperPrompt extends PromptElement<PromptProps, void> {
    render(state: void, sizing: PromptSizing) {
        return (
            <>
                <PersonalityPrompt {...this.props} />
                <TableSchemaPrompt {...this.props} />
                <UserMessage>
                    ## Query Details (User Input
                    
                    {this.props.userQuery}
                </UserMessage>
            </>
        );
    }
}
```

This `PromptElement` generated three messages:

* Define the chat participant's personality and any extra information and expectations about the participant.
* Create context about the table schema from our database
* The user's query (which contains the prompt from the user - the desired SQL query)

Oh, and that `this.props` object?  Yep - that's custom too.  When you define a `PromptElement` you have to also pass in an object containing properties about the prompt.  In our case, that's the user's query as well as the table schema.

### Pass this all into Copilot...

Ok, so we have our PromptElement defined.  It's now time to start implementing our handler and adding actual logic to it.

Our handler started to take shape:

``` typescript
const tableSchema = await getTables();

const {messages} = await renderPrompt(
    SqlHelperPrompt,
    { userQuery: request.prompt, tableSchema: tableSchema},
    { modelMaxPromptTokens: model.maxInputTokens },
    model
);

const chatResponse = await model.sendRequest(messages, {}, token);

for await (const fragment of chatResponse.text) {
    stream.markdown(fragment);
}
```

Ok, let's break this down.  First, we get our table schema.  Yes, we are getting this on every request.  Yes, we know this is bad.  We'll fix it later.

Next, we render our prompt using the `renderPrompt()` function.  This function takes a `PromptElementCtor` and the properties to pass into it, along with some other options.  We then send this prompt to Copilot using the `sendRequest()` function on our model, which sends the fully assembled prompt off to Copilot, and then we wait for the magic to happen and render the response to the chat window.

So, why a `for await` loop there?  Well, Copilot sends back a stream of messages, and we have to handle each message as it comes in.  That's why you'll see Copilot Chat (and other GenAI tools) slowly render text to the screen - it's sending back a stream of messages, not just one big message.

## Conclusion

And...  that's it!  Right?  Well, at this point we had a functional proof-of-concept.  And that's what we demo'd to the rest of the team.  We had a working chat participant that could connect to a database, get the schema, pass it along to Copilot, and generate a query based off of a user's prompt.  There are a few things we want to go back and improve upon:

1. Pre-flight the schema and only pass the relevant tables to Copilot.  This will help with the token limit issue.
2. Cache the schema so we don't have to get it on every request.
3. Don't prompt the user for the connection every time.  That's annoying.
4. Right now there's no concept of chat history, so we can't have a conversation with Copilot.  We'd like to add that in the future.
5. Right now we only take the table schema into account - as in the columns and their respective data types.  In the future, we'd also like to account for things like indexes, foreign keys, etc...  Also things like Views, Stored Procedures, etc...  And maybe even the data itself, though there's a lot of privacy concerns there.

All in all, this was a HUGE success, and we learned a ton.  This is proof that the Copilot Extensions ecosystem is going to be a game-changer for GitHub Copilot, and I really wanted to see just how far we could push this tool to enhance a very common ask I have from clients.  This is what Innovation Day is all about - trying something new, learning, and sharing that knowledge with others.  I'm excited to see where this project goes in the future, and I'm excited to see what other cool things my colleagues came up with today.  I'm truly blessed to work for such an awesome employer, and I can't wait to see what the future holds for us.
