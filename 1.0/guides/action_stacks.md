# Action Stacks

[Back to the Documentation index](../index.md)

While actions are being performed, Flint can maintain a list of “stacks” that track the history of user actions on a per-feature basis.

These stacks are scoped per Action Session and Feature, and support nested stacks. Each time the user performs an Action from a Feature that is not currently active in the current set of Action Stacks, a new stack will be created for that Session and Feature.

The result is that at any point in time you can browse the current active stacks, or log these out if you experience a crash or the user makes a support request.

You will get information about every action they performed in time order per feature, across all the sessions so you can see possible interactions between background and foreground tasks.

## Enabling Action Stacks

Doc coming soon

## Browsing active stacks with Flint UI

Doc coming soon

## Generating a debug report containing the stacks

Doc coming soon about how you can:

* Generate the full Flint report
* Generate just the Stacks report
* Generate the report with human readable or JSON formats

