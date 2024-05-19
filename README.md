# Carroted Bot API Protocol v1.2

The Carroted Bot API Protocol (usually shortened to Carroted Bot Protocol) is a protocol for communication between bots on text chat platforms like Discord. It is designed to facilitate standardized communication between bots without requiring them to use HTTP or other web protocols.

Benefits of this protocol include:
- No need to host a web server
- No IP address exposed or domain name required
- Everything happens conveniently on the chat platform, which can make use of the fact that the bot already has a WebSocket connection open to that
  - Bots essentially have a WebSocket connection to each other, and can thus provide status updates on the fly

## Request Format

Requests are sent in text messages with this simple format:

```
<bot ID>!api <command> <args>
```

For instance, here's a command to check a user's balance:

```
123456789!api balance 123456789
```
In the above example, the balance command is being sent to the bot with ID 123456789, and the argument is the user ID 123456789. Thus, it is requesting a bot to return its own balance.

Arguments can be in any format you wish, such as:
```
123456789!api payrequest 123456789 100 This is a cool message which doesn't need to be URL encoded or wrapped in quotes
```
In the above example, bot 123456789 is responsible for ensuring that the message is parsed correctly.

The argument format is intentionally vague and left up to the API/bot developer to decide, as this is just a protocol. It's comparable to how in HTTP, URL routes can be whatever you want, and HTTP headers can be whatever you want. In the same way, the argument format is up to you.

Bots simply send requests with string concatenation like:
```js
communicationChannel.send(`${botID}!api balance ${userID}`);
```
The client is responsible for sanitizing the input before sending the request.

## Response Format

Responses should be directly in the message content without any embeds or special formatting. No codeblocks should be used. JSON is heavily recommended except for single values, such as a balance request.

Responses **must be sent as a Reply** in order to ensure that responses aren't mixed up. Bots should ignore responses that weren't sent by the bot that the request was sent to.

Clients should expect that they can receive multiple responses to a single request, since the bot may send status updates or other messages in between responses.

Example:
```
Bot 1234: 5678!api balance 1234
Bot 5678: 100

Bot 1234: 5678!api someasynccommand
Bot 5678: {"status": "started"}
Bot 5678: {"type": "success", "message": "Done!"}
```
Example with JSON:
```
Bot 1234: 5678!api balance 1234
Bot 5678: {"balance": 100}
```
Codeblock syntax should not be used and is non-compliant with the protocol, as it introduces unnecessary complexity just for the sake of looking better for humans, while the protocol is designed for bots to communicate with each other efficiently.

Code example:
```js
if(message.content.startsWith(`${botID}!api balance`)) {
    const userID = message.content.split(" ")[2];
    const balance = await getBalance(userID);
    message.reply(balance); // No codeblocks, and no JSON since it's just a single value. We reply so that the client knows that this is a response to the request.
}
```

## Error Handling

Errors should be sent as a response with the following format:
```json
{
    "type": "error",
    "code": 404,
    "message": "User not found"
}
```
Note that codeblocks should not be used, we just used one here for formatting purposes.

Here's an example of how this could play out:
```
Bot 1234: 5678!api balance 1234
Bot 5678: {"balance": 100}
Bot 1234: 5678!api balance 9999
Bot 5678: {"type": "error", "code": 404, "message": "User not found"}
Bot 1234: 5678!api somethingelse
Bot 5678: {"type": "error", "code": 500, "message": "Internal server error"}
Bot 1234: 5678!api somethingelse
Bot 5678: {"type": "error", "code": 429, "message": "Too many requests"}
```

We recommend using HTTP status codes such as 404, 403, 500, etc. for maximum compatibility with other bots. You can find a list of them here: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status

For instance, other bots could be programmed to handle 404 errors by sending a message to the user saying that the user was not found, or 403 errors by sending a message to the user saying that they do not have permission to perform that action. If you instead use non-standard error codes, other bots will not be able to handle them without special code.

You can also easily find HTTP status codes with your favorite search engine, like "http status code for too many requests".

In the above example, we showed a "Too many requests" error. Ratelimiting isn't part of the protocol, but it's recommended that bots implement it to prevent abuse. If a bot receives too many requests, it should respond with a 429 error.

## Notes

- Users should be able to run API commands, though they can be given less API permissions than other bots. In this case, the bot should respond with error code 403.
- Running `<bot ID>!api` with no command should return API documentation or a list of commands.
