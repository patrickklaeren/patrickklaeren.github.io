---
layout: post
title: Migrating from Microsoft Teams to Discord
permalink: /2022/migrating-from-teams-to-discord
---

Communication is a mess in almost any human setting. Trying to get everyone onto the same page is a feat not many people can manage. You end up with a scale of free for all to micromanaging, where the latter usually alienates people but, in my experience, sometimes yields the best results. Thousands, if not millions, of solutions, or at least attempted, solutions have tried to conquer the Discord (heh, get it?) between people at work, home or in any other setting.

Today the options usually distil down to Slack, Microsoft Teams, Rocket Chat, or another Slack-like competitor. These apps have become "necessary evils" even though they all individually provide massive bottlenecks and drawbacks. This was particularly evident during the pandemic and lockdowns where these applications thrived and work moved to text-based back and forth conversations, sometimes arguments. We experienced this ourselves, moving first from Slack to Teams, then eventually to Discord – a platform that is not the first choice for any "professional" team, and a no-go for any company that requires strict compliance with auditing checks and the likes. Though, our choice of Discord has faired extremely well, in a world where remote colleagues now are able to sit in voice channels and people can drop in and out when they wish. Gone are the days of scheduled meetings and tight requirements for a channel to exist. Discord's loose approach to collaboration and communication makes for an inviting difference in an otherwise samey corporate stiffness.

Mobilising an entire company to use any new piece of software is a new learning experience every time. Discord was no different. When you first move, you'll be in the deep end. Integrations for anything "real" like Jira, GitHub and the like are slim. Webhooks are natively supported, and you can have GitHub post to Discord for events, but anything granular and you're left to wonder what could be if Discord did position itself as a Slack competitor. In some regards I am thankful Discord isn’t positioning itself as such because its platform remains more open and freer to developers like me who choose to build an entire devops experience off Discord.

In April 2020 we started our Discord adventure and more than two years on we now have our own Discord bot that happily sits listening for events, posting to the channels where the event is relevant to and keeping every single person, on the Discord, in the know. Need a new metric? No problem. Let's get that into the Discord bot.

Here's how I achieved it.

In the .NET Core world, ASP.NET is a wonderfully equipped platform to build almost any application on top of almost any hosting provider. Back in the days of ASP.NET MVC (Framework) you'd often ask yourself, "can I host my ASP.NET app here" and the answer would almost without a doubt be no, no you cannot. With the advent of .NET Core running across Windows, Mac and Linux, ASP.NET Core has become a verstaile enough platform to be able to host anything from niche Discord bots to multi-million simultaneous connections to a website or API. The foundation of the bot was therefore going to be ASP.NET Core, with the Discord bot as a [hosted service](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-6.0&tabs=visual-studio). While I didn't need a website *yet*, ASP.NET Core would provide the benefits of allowing that to happen in future, while also allowing me to just *turn that functionality off* in a crude sense.

There are a number of open source .NET libraries that help you interface with Discord's API. While [Discord.NET](https://github.com/discord-net/Discord.Net) and [DSharpPlus](https://github.com/DSharpPlus/DSharpPlus) exist, the choice for our internal bot is [Remora.Discord](https://github.com/Remora/Remora.Discord). The reason for this is that it is the lowest-level available library that directly tries to mimic the structure and API surface of Discord's own API documentation. There are no expensive or convoluted abstractions, which brings benefits to both the consumers of Remora and the developers maintaining the library.

The project structure isolates Discord bot logic from the ASP.NET Core project. This allows us to substitute in another host for the bot if the need ever arises - though, I doubt that will ever be the case.

```
src/
    Contoso.API
    Contoso.DiscordBot
```

Is a crude depiction of the project structure, where `Contoso.API` is our ASP.NET Core host and `Contoso.DiscordBot` is a .NET Core class library.

In `Contoso.DiscordBot` we will want to define our bot client, which will eventually be started by our ASP.NET Core host. We will also configure our Discord bot with an options class. Not everything defined here will be relevant for your own set up.

```cs
public class DiscordBotOptions
{
    public static string Key => "Discord";
    public ulong GuildId { get; set; }
    public ulong VoiceChannelId { get; set; }
    public ulong VoiceRoleId { get; set; }
}
```

These options will be available bot-wide wherever required. We have configurable IDs for channels in Discord, which are referred to as Snowflakes. In the `appsettings.json`, we have the following configurations defined:

```json
  "Discord": {
    "Token": "",
    "GuildId": "",
    "VoiceRoleId": "",
    "VoiceChannelId": ""
  }
```

It's important to note that these configurations have to be hydrated from the hosting app, i.e. the ASP.NET Core application. On startup of the host, we push the ASP.NET Core configuration into the Discord bot. We have a `IBotClient`, and implementation of said interface, to contain the start up and run logic of the bot.

```cs
public interface IBotClient
{
    Task Run(CancellationToken cancellationToken);
}

public class BotClient : IBotClient
{
    private readonly ILogger<BotClient> _logger;
    private readonly IOptions<DiscordBotOptions> _options;
    private readonly SlashService _slashService;
    private readonly DiscordGatewayClient _discordGatewayClient;
    
    public BotClient(
        ILogger<BotClient> logger,
        IOptions<DiscordBotOptions> options,
        SlashService slashService, 
        DiscordGatewayClient discordGatewayClient
    )
    {
        _logger = logger;
        _options = options;
        _slashService = slashService;
        _discordGatewayClient = discordGatewayClient;
    }

    public async Task Run(CancellationToken cancellationToken)
    {
        await RegisterSlashCommands(cancellationToken);

        var runResult = await _discordGatewayClient.RunAsync(cancellationToken);

        if (!runResult.IsSuccess)
        {
            switch (runResult.Error)
            {
                case ExceptionError exe:
                {
                    _logger.LogError(exe.Exception,"Exception during gateway connection: {ExceptionMessage}", exe.Message);
                    break;
                }
                case GatewayWebSocketError:
                case GatewayDiscordError:
                {
                    _logger.LogError("Gateway error: {Message}", runResult.ToString());
                    break;
                }
                default:
                {
                    _logger.LogError("Unknown error: {Message}", runResult.ToString());
                    break;
                }
            }
        }
    }

    private async Task RegisterSlashCommands(CancellationToken cancellationToken)
    {
        var checkSlashSupport = _slashService.SupportsSlashCommands();

        if (!checkSlashSupport.IsSuccess)
        {
            _logger.LogWarning("The registered commands of the bot don't support slash commands: {Reason}", checkSlashSupport.ToString());
        }
        else
        {
            var updateSlash = await _slashService.UpdateSlashCommandsAsync(new Snowflake(_options.Value.GuildId), cancellationToken);

            if (!updateSlash.IsSuccess)
            {
                _logger.LogWarning("Failed to update slash commands: {Reason}", updateSlash.ToString());
            }
        }
    }
}
```

The `IBotClient` interface is used within the background service implementation in the ASP.NET Core host app, which becomes a hosted service.

```cs
public class BotHostedService : BackgroundService
{
    private readonly IBotClient _botClient;
    private readonly ILogger<BotHostedService> _logger;

    public BotHostedService(IBotClient botClient, ILogger<BotHostedService> logger)
    {
        _botClient = botClient;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            _logger.LogInformation("Starting bot...");

            await _botClient.Run(stoppingToken);

            _logger.LogInformation("Stopped bot!");
        }
        catch (Exception e)
        {
            _logger.LogCritical(e, "Failed starting Discord bot");
            throw;
        }
    }
}
```

On startup, two methods are invoked in the ASP.NET Core app:

```cs
services
    .AddBotClient(builder.Configuration)
    .AddHostedService<BotHostedService>()
```

The former is an extension method defined in the Discord bot class library which isolates the logic, allowing for the configuration of the bot client to happen anywhere a Microsoft Extensions Dependency Injection container is available.

```cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddBotClient(this IServiceCollection collection, IConfiguration configuration)
    {
        var token = configuration["Discord:Token"];

        return collection
            .AddDiscordGateway(_ => token)
            .AddDiscordCommands(true)
            .Configure<DiscordBotOptions>(configuration.GetSection(DiscordBotOptions.Key))
            .Configure<DiscordGatewayClientOptions>(o =>
            {
                o.Intents |= GatewayIntents.DirectMessageReactions;
                o.Intents |= GatewayIntents.GuildMessageReactions;
                o.Intents |= GatewayIntents.GuildPresences;
                o.Intents |= GatewayIntents.GuildVoiceStates;
            })
            .AddCommandGroup<GitHubCommandGroup>()
            .AddResponder<GitHubLinkResponder>()
            .AddScoped<CommandResponder>()
            .AddTransient<IBotClient, BotClient>();
    }
}
```

This is where most of the configuration magic happens, we grab the Discord token and set up intents and Remora concepts, such as Responders and Command Groups. Responders are types that, well, respond to events in the Discord guild(s) the bot is in, for example, a message is received. Command groups are types that declare available commands in Discord, typically available via a slash (or `/`) syntax.

One of the annoyances of any chat platform includes discussing a particular issue or pull request. I added a responder that listens for any received message and that messages' contents includes a GitHub issue/pull request reference. I decided to denote the syntax as `#` followed by just numbers. This way if someone referenced `123`, we wouldn't trigger the bot, but if someone referenced `#123`, the bot would trigger and look for an issue or pull request with ID 123. Regex to the rescue!

```cs
public class GitHubLinkResponder : IResponder<IMessageCreate>
{
    private readonly ILogger<GitHubLinkResponder> _logger;
    private readonly IDiscordRestChannelAPI _channelApi;

    public GitHubLinkResponder(ILogger<GitHubLinkResponder> logger, 
        IDiscordRestChannelAPI channelApi)
    {
        _logger = logger;
        _channelApi = channelApi;
        _gitHubApiService = gitHubApiService;
        _repositorySubscriptionService = repositorySubscriptionService;
        _userService = userService;
    }

    public async Task<Result> RespondAsync(IMessageCreate gatewayEvent, CancellationToken ct = new())
    {
        var processEmbeds = false;

        if (GitHubLinkHelper.ContainsGitHubIssueLink(gatewayEvent.Content))
        {
            await HandleIssueLink(gatewayEvent, ct);
            processEmbeds = true;
        }

        if (GitHubLinkHelper.ContainsGitHubPullRequestLink(gatewayEvent.Content))
        {
            await HandlePullRequestLink(gatewayEvent, ct);
            processEmbeds = true;
        }

        if (processEmbeds 
            && !gatewayEvent.Embeds.Any())
        {
            await _channelApi.EditMessageAsync(gatewayEvent.ChannelID,
                gatewayEvent.ID,
                flags: MessageFlags.SuppressEmbeds,
                ct: ct);
        }

        return Result.FromSuccess();
    }
}
```

This is largely a stripped down implementation - however by implementing `IResponder<IMessageCreate>` on our class, we will receive any message sent to a channel in Discord where our bot is present. Every message is run through our `GitHubLinkHelper`, which scans the contents of the message against our Regex for GitHub ID syntax. If matched, we process the message and post it to Discord with context fetched from the GitHub API.

So why do I host the bot in an ASP.NET Core container? It allows me to post anything to a defined endpoint and have it relayed to Discord. The two concepts completely isolated, where I could route any API request to another bot or application. Since the Discord bot is configured within ASP.NET Core, I have access to the very same bot client that is running and connected to Discord. It is simply a case of getting the instance of the bot client and posting to the relevant channel.

```cs
[HttpPost("ondeployment")]
public async Task<IActionResult> OnDeployment(AzureDevopsDeploymentModel model)
{
    _logger.LogInformation("Received {MethodName}", nameof(OnDeployment));

    try
    {
        var dto = model.CreateDto();

        var channelIds =
            await _repositorySubscriptionService.GetDiscordChannelIdsForRepositoryName(dto.ProjectName);

        foreach (var channelId in channelIds)
        {
            await _discordPostService.PostAzurePipelinesDeployment(channelId, dto);
        }
    }
    catch (Exception e)
    {
        _logger.LogCritical(e, "Failed {MethodName}", nameof(OnDeployment));
    }

    return Ok();
}
```

The implementation of such a tool for internal use is very specific to our use cases. I fixed annoyances and made quality of life improvements to what everyone likes to refer to as chat ops. Today we have a Discord bot that integrates with our GitHub, Sentry, Azure Devops and SEQ.