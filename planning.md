# FitFindr — planning.md

---

## Tools

### Tool 1: search_listings

**What it does:**
Searches through the mock listings dataset to find items that match the user's description, size preference, and budget. Returns the top matching items sorted by how well they match the description.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): A text description of what the user is looking for, like "vintage graphic tee" or "black leather jacket"
- `size` (str): he clothing size the user needs, like "M" or "XL", and "W28" or "One Size" 
- `max_price` (float): The highest price the user is willing to pay

**What it returns:**
Returns a list of matching listing dictionaries. Each dictionary includes id, title, description, category, style_tags, size, condition, price, colors, brand, and platform. If no matches found, returns an empty list.

**What happens if it fails or returns nothing:**
The agent tells the user "No items found matching [description] in size [size] under $[max_price]. Try removing the size filter or increasing your budget." Then the agent stops and does not call any other tools

---

### Tool 2: suggest_outfit

**What it does:**
Takes a new item and the user's existing wardrobe, then generates one or more outfit combinations that work well together. Includes specific styling tips like how to tuck or roll sleeves.

**Input parameters:**

- `new_item` (dict): A listing dictionary representing the item the user just found, containing fields like title, style_tags, colors, and category
- `wardrobe` (dict): A dictionary with an 'items' key containing a list of wardrobe item dictionaries that the user already owns

**What it returns:**
Returns a string with outfit suggestions. If wardrobe is empty, returns general styling advice instead of crashing.

**What happens if it fails or returns nothing:**
If wardrobe is empty, the tool still returns styling advice like "This tee would look great with any dark jeans or layered under a jacket" without referencing specific items. Never returns an empty string.
---

### Tool 3: create_fit_card

**What it does:**
Turns an outfit suggestion into a short, shareable Instagram-style caption that sounds like something a real person would post. Each fit card should be unique based on the items and style.

**Input parameters:**

- `outfit` (str): The outfit suggestion returned from suggest_outfit, containing description, styling_tips, and vibe

- `new_item` (dict): The listing dictionary for the item being styled

**What it returns:**
Returns a 2-4 sentence string usable as a social media caption. Includes the item name, price, and platform naturally.

**What happens if it fails or returns nothing:**
If outfit string is empty or missing, returns an error message string like "Could not generate fit card because outfit suggestion was missing." Does not raise an exception.

---

### Additional Tools (if any)
NA
---

## Planning Loop

**How does your agent decide which tool to call next?**
The agent follows a linear workflow with one conditional branch. Here is the exact logic:

First, call search_listings with the user's description, size, and max_price. Check if the results list is empty. If it is empty, set session["error"] to a message telling the user no items were found and return immediately. Do not call any other tools.

If search_listings returns results, take the first item from the list and store it in session["selected_item"]. Then call suggest_outfit with selected_item and the user's wardrobe. Store the result in session["outfit_suggestion"].

Then call create_fit_card with session["outfit_suggestion"] and session["selected_item"]. Store the result in session["fit_card"].

The agent knows it is done after create_fit_card returns. It then returns the full session dictionary to be displayed to the user.

The agent never loops back to previous tools. The only decision point is whether search_listings found any results.

---

## State Management

**How does information from one tool get passed to the next?**
The agent uses a session dictionary that persists across all tool calls in one user interaction. The session dictionary starts with these keys all set to None: selected_item, outfit_suggestion, fit_card, error.

After search_listings runs, the agent extracts the first result and stores it as session["selected_item"]. If there are no results, session["error"] gets set instead.

When suggest_outfit runs, it receives session["selected_item"] as its new_item parameter. It returns a string that gets stored as session["outfit_suggestion"].

When create_fit_card runs, it receives session["outfit_suggestion"] as the outfit parameter and session["selected_item"] as the new_item parameter. It returns a string that gets stored as session["fit_card"].

At the end, the session dictionary contains all the information needed to display results to the user. The user does not have to re-enter any data between tool calls.

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Agent says "I couldn't find any [description] in size [size] under $[max_price]. Try removing the size filter or increasing your budget." Then stops and returns without calling other tools.|
| suggest_outfit | Wardrobe is empty | Tool still returns styling advice like "Since your wardrobe is empty, here is how you could style this: [general advice]. Add some items to your wardrobe for more personalized suggestions."|
| create_fit_card | Outfit input is missing or incomplete | Tool returns an error message string: "Could not generate fit card because outfit suggestion was missing." Agent shows this message to the user instead of a caption. |

---

## Architecture
User gives query (description, size, max_price, wardrobe)

        |
        v

run_agent()

        |
        v

search_listings(description, size, max_price)

        |
        v

    CHECK RESULTS
        |
        +-------[no results]------> session["error"] = "No items found"
        |                               |
        |                               v
        |                         Return to user (STOP)
        |
        +-------[has results]----> session["selected_item"] = results[0]
                                        |
                                        v
                                suggest_outfit(selected_item, wardrobe)
                                        |
                                        v
                                session["outfit_suggestion"] = result string
                                        |
                                        v
                                create_fit_card(outfit_suggestion, selected_item)
                                        |
                                        v
                                session["fit_card"] = result string
                                        |
                                        v
                                Return session to user

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
For search_listings, I will use Claude. I will give it the Tool 1 section from planning.md with the specific inputs, return format, and failure behavior. I will also give it the load_listings function from data_loader.py. I will ask Claude to implement search_listings that filters by checking if the description appears in title or description fields, matches size exactly, and keeps price less than or equal to max_price. I need to tell Claude to make size matching case insensitive and to handle size=None or max_price=None correctly. Before running, I will check that the generated code returns an empty list for no matches. Then I will test with three queries: one that should find matches, one with no matches, and one with a high price filter that returns everything. After Claude gives me the code, I will also ask you to review it before I run it.

For suggest_outfit, I will use Claude. I will give it the Tool 2 section and the wardrobe_schema.json structure. I will ask Claude to implement a function that uses _get_groq_client() from tools.py to call the LLM. I will specify that the prompt should include the new_item details like title, style_tags, colors, and category, plus the wardrobe items if available. I will tell Claude to handle empty wardrobe by returning general advice without crashing. I will verify that the output is a non-empty string and that it mentions specific wardrobe items when wardrobe is not empty. I will ask you to review the prompt structure Claude suggests before I implement.

For create_fit_card, I will use Claude. I will give it the Tool 3 section and ask it to implement a function using the Groq client. I will specify that the LLM temperature should be set to 0.8 or higher to get varied captions each time. I will ask for a prompt that produces casual Instagram-style captions with emojis. I will verify that the output is a string between 15 to 40 words and includes the price and platform naturally. I will test the fallback by feeding it an empty outfit string and confirming it returns an error message without crashing. I will ask you to check Claude's temperature recommendation before I finalize.

**Milestone 4 — Planning loop and state management:**
I will use Claude to implement the main agent orchestration. I will give it the Planning Loop section, State Management section, and the Architecture diagram from planning.md. I will also give it the three completed tool functions from tools.py. I will ask Claude to write a run_agent function in agent.py that follows the linear flow with conditional branching after search. I will specify that the session dictionary must track selected_item, outfit_suggestion, fit_card, and error_message. Before running, I will manually trace through the logic using the example query to make sure the state flows correctly between tools. I will test with a successful search path and a failed search path to verify error handling stops the agent early. I will also test that suggest_outfit and create_fit_card are not called when search_listings returns an empty list. After Claude provides the code, I will ask you to review the planning loop logic to make sure it matches my spec

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1: Search**
FitFindr calls search_listings("vintage graphic tee", size="M", max_price=30.0). Returns top 3 matching listings and picks the most relevant one. If no results found, tells user to try different filters and stops

**Step 2: Suggest Outfit**
Using the item returned from search, FitFindr calls suggest_outfit(new_item=<band tee>, wardrobe=<user's wardrobe>). Returns styling advice that incorporates the user's existing clothes. Works even with empty wardrobe by suggesting general pairings.

**Step 3: Create Fir Card**
With the outfit suggestion, FitFindr calls create_fit_card(outfit=<suggestion>, new_item=<band tee>). Generates a short, shareable Instagram-style caption. On rare failure, returns a simple fallback template.

**Final output to user:**
Shows the found item, styling advice, and fit card all together in one response.
