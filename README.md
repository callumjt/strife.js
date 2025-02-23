# strife.js

A Discord bot framework.

## Installation

Install alongside discord.js:

```sh
npm install discord.js strife.js
```

strife.js officially supports discord.js version 14.9 and above.

## Login

Login to your Discord bot by calling `login`:

```js
import { GatewayIntentBits } from "discord.js";
import { login } from "strife.js";
import path from "node:path";
import url from "node:url";

await login({
	clientOptions: { intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMembers] },
	modulesDirectory: path.resolve(path.dirname(url.fileURLToPath(import.meta.url)), "./modules"),
});
```

Once `login` has been called, you may import `client` from anywhere in your app:

```js
import { client } from "strife.js";
import { Client } from "discord.js";

client instanceof Client; // true
```

Note that `client` is `undefined` before `login` is called, and this behavior is not accounted for in the types, so plan appropriately.

### Configuration

#### `clientOptions`

Type: `ClientOptions`

Options to pass to discord.js. As in discord.js, the only required property is `intents`. strife.js has some defaults on top of discord.js's:

-   `allowedMentions` is set to only ping users by default (including replied users) to avoid accidental mass pings
-   `failIfNotExists` is set to `false` to return `null` instead of erroring in certain cases
-   `partials` is set to all available partials to avoid missed events

Of course, these are all overridable.

#### `modulesDirectory`

Type: `string`

The directory to import modules from. See [Usage](#usage) for detailed information. It is recommended to set this to `path.resolve(path.dirname(url.fileURLToPath(import.meta.url)), "./modules")`.

#### `botToken`

Type: `string | undefined`

The token to sign in to Discord with. Defaults to `process.env.BOT_TOKEN`

#### `commandErrorMessage`

Type: `string | undefined`

The message displayed to the user when commands fail. Omit to use Discord's default `❗ The application did not respond`.

#### `handleError`

Type: `((error: any, event: ClientEvent | RepliableInteraction) => Awaitable<void>) | undefined`

Called when an error occurs in discord.js or any event, component, or command handler. Defaults to `console.error`.

#### `defaultCommandAccess`

Type: `boolean | Snowflake | Snowflake[]`

The default value of [a command's `access` field](#access).

## Usage

It is strongly recommended to use this framework with TypeScript. In the future, this framework will provide more powerful dynamic types for your bots.

Every file in [`modulesDirectory`](#modulesdirectory), and every `index.js` in subdirectories of `modulesDirectory` will be automatically imported after logging in, but before commands are registered. It is recommended to only call the below functions in those files.

### Commands

Use the `defineChatCommand` function to define a basic chat command.

```js
import { defineChatCommand } from "strife.js";

defineChatCommand(
	{ name: "ping", description: "Ping!" },

	async (interaction) => {
		await interaction.reply("Pong!");
	}
);
```

Use the `restricted: true` option to deny members permission to use the command, and require guild admins to explicitly set permissions via `Server Settings` -> `Integrations`.

#### Options

You can specify options for commands using the `options` property. This is a key-value pair where the keys are option names and the values are option details.

```js
import { defineChatCommand } from "strife.js";
import { ApplicationCommandOptionType, User } from "discord.js";

defineChatCommand(
	{
		name: "say",
		description: "Send a message",

		options: {
			message: {
				type: ApplicationCommandOptionType.String,
				description: "Message content",
				maxLength: 2000,
				required: true,
			},
		},
	},

	async (interaction, options) => {
		// code here...
	}
);
```

`type` (`ApplicationCommandOptionType`) and `description` (`string`) are required on all options. `required` (`boolean`) is optional, defaulting to `false`, but is allowed on all types of options.

Some types of options have additional customization fields:

-   `Channel` options allow `channelTypes` (`ChannelType[]`) to define allowed channel types for this option. Defaults to all supported guild channel types.
-   `Integer` and `Number` options allow `minValue` and/or `maxValue` to define lower and upper bounds for this option respectively. Defaults to `-2 ** 53` and `2 ** 53` respectively.
-   `String` commands allow a few additional fields:
    -   `choices` (`Record<string, string>`) to require users to pick values from a predefined list. The keys are the descriptions displayed to the users and the values are what is passed to your bot. No other additional fields are allowed for this option when using `choices`.
    -   `minLength` (`number`) and/or `maxLength` (`number`) to define lower and upper bounds for this option's length respectively. Defaults to `0` and `6_000` respectively.
    -   `autocomplete` (`(interaction: AutocompleteInteraction) => ApplicationCommandOptionChoiceData<string>[]`) to give users dynamic choices.
        -   Use `interaction.options.getFocused()` to get the value of the option so far. You can also use `interaction.options.getBoolean()`, `.getInteger()`, `.getNumber()`, and `.getString()`. Other option-getters will not work, use `interaction.options.get()` instead.
        -   Return an array of choice objects. It will be truncated to fit the 25-item limit automatically.
        -   Note that Discord does not require users to select values from the options, so handle values appropriately.
        -   Also note that TypeScript cannot automatically infer the value of the type parameter, however, it will error if you set it incorrectly.

To retrieve option values at runtime, you can utilize the second `options` parameter of the command handler. You can always use Discord.JS's `interaction.options` API, however, the `options` parameter is a key-value object of options. That is often simpler to use and has better types when using TypeScript.

#### Subcommands

You can specify subcommands using the `defineSubcommands` function. Define subcommands using the `subcommands` property, which is a key-value pair where the keys are subcommand names and the values are subcommand details. Subcommands must have `name`s and `description`s and may have `options`.

```js
import { defineSubcommands } from "strife.js";

defineSubcommands(
	{
		name: "xp",
		description: "Commands to view users’ XP amounts",

		subcommands: {
			rank: { description: "View your XP rank", options: {} },
			top: { description: "View the server XP leaderboard", options: {} },
		},
	},

	async (interaction, { subcommand, options }) => {
		// code here...
	}
);
```

The root command description is not displayed anywhere in Discord clients, but it is still required by the Discord API. Subcommands support options in the same way as regular commands.

When using subcommands, the second argument to the handler is an object with the properties `subcommand` (`string`) and `options` (key-value pair as in `defineChatCommand`). In order for this parameter to be correctly typed, all subcommands must have `options` set, even if just to an empty object.

#### Subcommand Groups

You can specify subcommand groups using the `defineSubGroups` function. Define subgroups using the `subcommands` property, which is a key-value pair where the keys are subgroup names and the values are subgroup details. Subgroups must have `name`s, `description`s, and `subcommands`. Subcommands must have `name`s and `description`s and may have `options`.

```js
import { defineSubGroups } from "strife.js";

defineSubGroups(
	{
		name: "foo",
		description: "...",

		subcommands: {
			bar: {
				description: "...",
				subcommands: {
					baz: { description: "...", options: {} },
				},
			},
		},
	},

	async (interaction, { subcommand, subGroup, options }) => {
		// code here...
	}
);
```

The root command description and subgroup descriptions are not displayed anywhere in Discord clients, but they are still required by the Discord API. Subcommands support options in the same way as regular commands.

When using subcommands, the second argument to the handler is an object with the properties `subcommand` (`string`), `subGroup` (`string`), and `options` (key-value pair as in `defineChatCommand`). In order for this parameter to be correctly typed, all subcommands must have `options` set, even if just to an empty object.

Mixing subgroups and subcommands on the same level is not currently supported.

#### Menu Commands

You can specify menu commands using the `defineMenuCommand` function.

```js
import { defineChatCommand } from "strife.js";
import { ApplicationCommandType } from "discord.js";

defineMenuCommand(
	{ name: "User Info", type: ApplicationCommandType.User },

	async (interaction) => {
		// code here...
	}
);
```

Message context menu commands are also supported with `ApplicationCommandType.Message`.

#### Access

By default, commands are allowed in all guilds plus DMs.

To change this behavior, you can set `defaultCommandAccess` when logging in. Pass in `Snowflake | Snowflake[]` to only define commands in specified guilds, `false` to define them in every guild but no DMs, or `true` to use the default configuration. When using TypeScript, it is necessary to augment the `DefaultCommandAccess` interface when changing this. To do that, add the following in a new `.d.ts` file:

```ts
declare module "strife.js" {
	export interface DefaultCommandAccess {
		inGuild: true;
	}
}
```

Commands also support a root-level `access` option to override this on a per-command basis. It supports the same options, with the addition of `@default` in the array of `Snowflake`s to extend the default guilds. `@default` is not available if `defaultCommandAccess` is unset or is set to a boolean.

#### Augments

You can define custom command properties by using augments (advanced usage):

```ts
declare module "strife.js" {
	export interface AugmentedMenuCommandData<
		InGuild extends boolean,
		Context extends MenuCommandContext
	> {}

	export interface AugmentedRootCommandData<
		InGuild extends boolean,
		Options extends RootCommandOptions<InGuild>
	> {}
	export interface AugmentedSubcommandData<
		InGuild extends boolean,
		Options extends SubcommandOptions<InGuild>
	> {}
	export interface AugmentedSubGroupsData<
		InGuild extends boolean,
		Options extends SubGroupsOptions<InGuild>
	> {}

	export interface AugmentedChatCommandData<InGuild extends boolean> {}
	export interface AugmentedCommandData<InGuild extends boolean> {}
}
```

### Events

Use the `defineEvent` function to define an event.

```js
import { defineEvent } from "strife.js";

defineEvent("messageCreate", async (message) => {
	// code here...
});
```

Note that since [all partials are enabled by default](#clientoptions), it is necessary to [handle partials accordingly](https://discordjs.guide/popular-topics/partials.html#enabling-partials).

You are allowed to define multiple listeners for the same event. Note that listener execution order is not guaranteed.

#### Pre-Events

Pre-events are a special type of event listener that executes before other listeners. They must return `Awaitable<boolean>` that determines if other listeners are executed or not. They are defined with the `defineEvent.pre` function:

```js
import { defineEvent } from "strife.js";

defineEvent.pre("messageCreate", async (message) => {
	// code here...
	return true;
});
```

Remember to return a boolean to explicitly say whether execution should continue.

A use case for this would be an automoderation system working alongside an XP system. The automoderation system could define a pre-event to delete rule-breaking messages and return `false` so users do not receive XP for rule-breaking messages.

You are only allowed to define one pre-event for every `ClientEvent`. You can define a pre-event with or without defining normal events.

### Components

Use the `defineButton`, `defineModal`, and `defineSelect` functions to define a button, a modal, and a select menu respectively:

```js
import { defineButton } from "strife.js";

defineButton("foobar", async (interaction, data) => {
	// code here...
});
```

The button ID (`"foobar"` in this example) and the `data` parameter of the callback function are both taken from the button's `customId`. For example, if the `customId` is `"abcd_foobar"`, then the callback for the `foobar` button will be called and the `data` parameter will have a value of `"abcd"`.

The button `data` may not have underscores but the `id` may. For example, a `customId` of `"foo_bar_baz"` will result in an `id` of `"bar_baz"` and the `data` `"foo"`. You can also omit the data from the `customId` altogether - a `customId` of `"_foobar"` will result in an `id` of `"foobar"` and the `data` `""`.

It is not required for all `customId`s to follow this format nor to have an associated handler. You are free to collect interactions in any other way you wish alongside or independent of strife.js.

`defineModal` works in the same way as `defineButton` but for modals.

`defineSelect` also works similarly to `defineButton` for select menus with one exception. By default, `defineSelect` collects all types of select menus. However, you can specify certain types of select menus to collect for better type safety, especially with TypeScript.

```js
import { defineSelect } from "strife.js";
import { ComponentType } from "discord.js";

defineSelect("foobar", ComponentType.StringSelect, async (interaction, data) => {
	// code here...
});
```

You can specify `SelectMenuType | SelectMenuType[]`. Note that this does not allow you to define multiple callbacks for the same ID but for different types. If a non-matching type is encountered, nothing will happen and you are expected to handle the interaction yourself.

## Style Guide

None of the following are requirements other than those using the word "must", however, they are all _strongly_ encouraged. An ESLint plugin to enforce this style guide may be created in the future.

It is recommended to call your [`modulesDirectory`](#modulesdirectory) `modules`. Each file or folder in `modulesDirectory` should be a different feature of your bot.

It is discouraged to import one module from another. Each module should work independently.

### File Modules

JavaScript files that lie directly inside `modulesDirectory` should be short - a few hundred lines at most. If they are any longer, make them [directory modules](#directory-modules) instead.

### Directory Modules

Subdirectories of `modulesDirectory` _must_ have an `index.js` file. This file should contain all the definitions for the module. In other words, `defineChatCommand`, `defineEvent`, etc. should only be used in the `index.js` of directory modules. If any callback is more than a few dozen lines long, it should instead be imported from another file in the same directory. If multiple files use the same values, i.e. constants, they should go into `misc.js` in that directory.

## Imports

This guide references the following imported values in inline code blocks:

```js
import {
	type ApplicationCommandOptionType
	type AutocompleteInteraction,
	type Awaitable,
	type ChannelType,
	type ClientOptions,
	type RepliableInteraction,
	type Snowflake,
	type SelectMenuType,
	ApplicationCommandType,
} from "discord.js";
import {
	type ClientEvent,
	defineButton,
	defineChatCommand,
	defineSubcommands,
	defineSubGroups,
	defineMenuCommand,
	defineEvent,
	defineModal,
	defineSelect,
} from "strife.js";
import path from "node:path";
import url from "node:url";
```

Values only referenced in multiline code blocks are not listed here as they are imported there.
