# Autocomplete

Autocomplete allows you to dynamically provide a selection of values to the user, based on their input, rather than relying on static choices. In this section we will cover how to add autocomplete support to your commands.

::: tip
This page is a follow-up to the [slash commands](/slash-commands/advanced-creation.md) section covering options and option choices. Please carefully read those pages first so that you can understand the methods used in this section.
:::

## Enabling autocomplete

To use autocomplete with your commands, *instead* of listing static choices, the option must be set to use autocompletion using <DocsLink section="builders" path="class/SlashCommandStringOption?scrollTo=setAutocomplete" type="method" />:

```js {9}
const { SlashCommandBuilder } = require('discord.js');

const data = new SlashCommandBuilder()
	.setName('guide')
	.setDescription('Search discordjs.guide!')
	.addStringOption(option =>
		option.setName('query')
			.setDescription('Phrase to search for')
			.setAutocomplete(true));
```

## Responding to autocomplete interactions

To handle an <DocsLink path="class/AutocompleteInteraction"/>, use the <DocsLink path="class/BaseInteraction?scrollTo=isAutocomplete"/> type guard to make sure the interaction instance is an autocomplete interaction. You can do this in a separate `interactionCreate` listener:

<!-- eslint-skip -->

```js
client.on(Events.InteractionCreate, interaction => {
	if (!interaction.isAutocomplete()) return;
	// do autocomplete handling
});
```

Or alternatively, by making a small change to your existing [Command handler](/creating-your-bot/command-handling.md) and adding an additional method to your individual command files.

The example below shows how this might be applied to a conceptual version of the `guide` command to determine the closest topic to the search input:

:::: code-group
::: code-group-item index.js
```js {4,13}
client.on(Events.InteractionCreate, async interaction => {
	if (interaction.isChatInputCommand()) {
		// command handling
	} else if (interaction.isAutocomplete()) {
		const command = interaction.client.commands.get(interaction.commandName);

		if (!command) {
			console.error(`No command matching ${interaction.commandName} was found.`);
			return;
		}

		try {
			await command.autocomplete(interaction);
		} catch (error) {
			console.error(error);
		}
	}
});
```
:::
::: code-group-item commands/guide.js
```js
module.exports = {
	data: new SlashCommandBuilder()
		.setName('guide')
		.setDescription('Search discordjs.guide!')
		.addStringOption(option =>
			option.setName('query')
				.setDescription('Phrase to search for')
				.setAutocomplete(true)),
	async autocomplete(interaction) {
		// handle the autocompletion response (more on how to do that below)
	},
	async execute(interaction) {
		// respond to the complete slash command
	},
};
```
:::
::::

The command handling is almost identical, but notice the change from `execute` to `autocomplete` in the new else-if branch. By adding a separate `autocomplete` function to the `module.exports` of commands that require autocompletion, you can safely separate the logic of providing dynamic choices from the code that needs to respond to the slash command once it is complete.

:::tip
You might have already moved this code to `events/interactionCreate.js` if you followed our [Event handling](/creating-your-bot/event-handling.md) guide too.
:::

### Sending results

The <DocsLink path="class/AutocompleteInteraction"/> class provides the <DocsLink path="class/AutocompleteInteraction?scrollTo=respond"/> method to send a response. Using this, you can submit an array of <DocsLink path="typedef/ApplicationCommandOptionChoiceData" /> objects for the user to choose from. Passing an empty array will show "No options match your search" for the user.

::: warning
Unlike static choices, autocompletion suggestions are *not* enforced, and users may still enter free text.
:::

The <DocsLink path="class/CommandInteractionOptionResolver?scrollTo=getFocused" /> method returns the currently focused option's value, which can be used to applying filtering to the choices presented. For example, to only display options starting with the focused value you can use the `Array#filter()` method, then using `Array#map()`, you can transform the array into an array of <DocsLink path="typedef/ApplicationCommandOptionChoiceData" /> objects.

```js {10-15}
module.exports = {
	data: new SlashCommandBuilder()
		.setName('guide')
		.setDescription('Search discordjs.guide!')
		.addStringOption(option =>
			option.setName('query')
				.setDescription('Phrase to search for')
				.setAutocomplete(true)),
	async autocomplete(interaction) {
		const focusedValue = interaction.options.getFocused();
		const choices = ['Popular Topics: Threads', 'Sharding: Getting started', 'Library: Voice Connections', 'Interactions: Replying to slash commands', 'Popular Topics: Embed preview'];
		const filtered = choices.filter(choice => choice.startsWith(focusedValue));
		await interaction.respond(
			filtered.map(choice => ({ name: choice, value: choice })),
		);
	},
};
```

### Handling multiple autocomplete options

To distinguish between multiple options, you can pass `true` into <DocsLink path="class/CommandInteractionOptionResolver?scrollTo=getFocused"/>, which will now return the full focused object instead of just the value. This is used to get the name of the focused option below, allowing for multiple options to each have their own set of suggestions:

```js {10-19}
module.exports = {
	data: new SlashCommandBuilder()
		.setName('guide')
		.setDescription('Search discordjs.guide!')
		.addStringOption(option =>
			option.setName('query')
				.setDescription('Phrase to search for')
				.setAutocomplete(true))
		.addStringOption(option =>
			option.setName('version')
				.setDescription('Version to search in')
				.setAutocomplete(true)),
	async autocomplete(interaction) {
		const focusedOption = interaction.options.getFocused(true);
		let choices;

		if (focusedOption.name === 'query') {
			choices = ['Popular Topics: Threads', 'Sharding: Getting started', 'Library: Voice Connections', 'Interactions: Replying to slash commands', 'Popular Topics: Embed preview'];
		}

		if (focusedOption.name === 'version') {
			choices = ['v9', 'v11', 'v12', 'v13', 'v14'];
		}

		const filtered = choices.filter(choice => choice.startsWith(focusedOption.value));
		await interaction.respond(
			filtered.map(choice => ({ name: choice, value: choice })),
		);
	},
};
```

### Accessing other values

In addition to filtering based on the focused value, you may also wish to change the choices displayed based on the value of other arguments in the command. The following methods work the same in <DocsLink path="class/AutocompleteInteraction"/>:

```js
const string = interaction.options.getString('input');
const integer = interaction.options.getInteger('int');
const boolean = interaction.options.getBoolean('choice');
const number = interaction.options.getNumber('num');
```

However, the `.getUser()`, `.getMember()`, `.getRole()`, `.getChannel()`, `.getMentionable()` and `.getAttachment()` methods are not available to autocomplete interactions. Discord does not send the respective full objects for these methods until the slash command is completed. For these, you can get the Snowflake value using `interaction.options.get('option').value`:

### Notes

- As with other application command interactions, autocomplete interactions must receive a response within 3 seconds. 
- You cannot defer the response to an autocomplete interaction. If you're dealing with asynchronous suggestions, such as from an API, consider keeping a local cache.
- After the user selects a value and sends the command, it will be received as a regular <DocsLink path="class/ChatInputCommandInteraction"/> with the chosen value.
- You can only respond with a maximum of 25 choices at a time, though any more than this likely means you should revise your filter to further narrow the selections.
