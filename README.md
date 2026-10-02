# tmp

The multi-turn tool test passed (13/13 checks). It first failed with an HTTP 400, and that failure matters for anyone integrating a client with this model.

## Result

The test is `/home/sailplane/k2-tests/multiturn_tools.py`. It runs a real loop: the model requests tools, my code executes them with fixed data, and the results go back to the model.

**Turn 1** asked for the weather in Paris and Tokyo, which is warmer, and the difference in °F.
- **Step 1:** the model made two parallel `get_weather` calls (Paris 18°C, Tokyo 26°C).
- **Step 2:** it made two parallel `convert_c_to_f` calls, using the temperatures from step 1. It got 64.4°F and 78.8°F.
- **Step 3:** it answered "Tokyo is warmer by 14.4°F", which is correct.

**Turn 2** asked about Berlin and referred to "the two cities we just compared".
- It called `get_weather` for Berlin only and did not re-fetch Paris or Tokyo, so it used the earlier context.
- It correctly said Berlin is colder and compared against both cities.

The test also confirmed a tool call can follow a tool result, parallel calls work, and `finish_reason` is correct at each step (`tool_calls`, then `stop`).

## The 400 error

My first run failed on the second request, after the first tool results were sent back. The server log said: `Chat template rejected the request: Assistant message is missing a thinking field.`

K2-Horizon's chat template (lines 942–961 of `chat_template.jinja`) raises an error if any assistant message in the history lacks a `reasoning_content`, `reasoning`, `think`, `think_fast` or `think_faster` field. My script was dropping the reasoning when it rebuilt the assistant messages. Echoing the server's `reasoning_content` back fixed it.

**This is the main thing to know before pointing other clients at this endpoint.** Agent frameworks and SDK wrappers that rebuild the message list from only `role`, `content` and `tool_calls` will hit this same 400 on the second step of any tool loop. I haven't tested a client that does this, and I haven't tried editing the template to relax the check.

## Caveats

- **Single run:** this was one run at temperature 1.0, not a reliability measurement.
- **Model-side arithmetic:** in turn 2 the model computed Berlin's °F value (53.6) and the differences itself instead of calling `convert_c_to_f`. The numbers are right, but my system prompt said never to guess values. For this model, a system prompt isn't a reliable way to force a tool call.
- **Loose string checks:** the answer checks are substring matches, so they confirm the content is plausible and don't verify exact wording.
- **Still untested:** the `json` and `xml_typed` tool-call formats, very long contexts, and several requests at once.

I can try the other tool-call formats, or look at relaxing the chat template so clients that drop `reasoning_content` still work.
