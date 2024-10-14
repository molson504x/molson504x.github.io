---
layout: post
title: RoastGPT - An AI-Powered Fantasy Football Weekly Recap Generator
date: 2024-10-02 10:00:00-0400
description: >
  I'm in a couple of Fantasy Football leagues, and one of my favorite parts of Fantasy Football is the trash talk. I wanted to build an AI that could roast my friends' teams for me. So I built did what any good developer would do - I wrote a script which uses an Azure OpenAI service to generate a recap of the week's matchups, complete with roasts for each team.  
img: ai-fantasy-football-roast/hero.png
fig-caption: A group of engineers bolting an extension onto a robot.
tags: [Azure, OpenAI, AI, GPT4-o, ChatGPT]
---

I'm in a couple of Fantasy Football leagues, and one of my favorite parts of Fantasy Football is the trash talk. In one of my leagues, one of my weekly responsibilities is creating a weekly matchup recap newsletter, and naturally I use that as an opportunity for some good old-fashioned trash talk.  Since I do so much stuff with AI, this year I decided to utilize ChatGPT to help roast the matchups.  While it was doing a good job starting out, I quickly discovered that using ChatGPT to generate roasts for each team was a bit of a pain - I had to manually take screenshots of the matchup box scores, provide metadata to the AI to have some additional context, and the "vision" aspect of GPT-4o wasn't always right so I had to constantly correct the AI.  It was good, but I wanted it to do better.  So I wrote a python script which uses an Azure OpenAI service to generate a recap of the week's matchups, complete with roasts for each team.  Just for fun, I named it RoastGPT.

## How it Works

RoastGPT uses Azure OpenAI to generate the weekly matchup recaps.  It uses a deployment of GPT-4o-mini to generate the text.  The script also calls out to ESPN's API to get various information about the fantasy league, such as the team names, the scores, and the player stats.  The script then generates a series of prompts to pass into the AI, which generates the text for the newsletter.  Sounds easy enough, right?  I've said that before, and well...  No.

## Step 1 - Get the Data

After doing some searching on Google, it turns out that ESPN Fantasy doesn't have a published API.  I could probably have crawled the site to see if I could pull in an OpenAPI spec, but I didn't know where they would have had an OpenAPI specification published.  Instead I did what any other good developer would do and let someone do the work for me.  I happened to find the [espn_api](https://github.com/cwendt94/espn-api) python package, which claimed to be able to pull in data from ESPN's Fantasy API.  After a couple of quick tests, I found that it does, indeed, work as advertised.  Yay!

Since the ESPN APIs weren't really intended to be used outside of ESPN, there is a weird setup step that's required for this library, especially if you want to use it to view _private_ league information.  You have to go get a couple of session cookies from a browser session and use those for authenticating to the API.  I'm not going to go into the details of how to do that here since [it is documented](https://github.com/cwendt94/espn-api/discussions/150) but just know it is a required step.

## Step 2 - Generate the Prompts

Prompts are the way we interface with GenAI systems, such as ChatGPT.  It's how we tell it what to do, provide examples and details for what we expect it to do, and how we pass any relevant data into the AI to be processed.  For RoastGPT, I generate a series of prompts - one for each matchup and then one for an "intro" and one for a "outro".  The prompts are generated by pulling in the data from the ESPN API and then formatting it in a way that the API can understand.  This is where life gets a little tricky.

### Formatting the Data

The ESPN API returns a TON of data, and some of it includes some duplicate data.  Most of it isn't needed.  To generate a Fantasy Football recap, you really only need some key pieces of data - the team names, scores, and lineup stats.  I also wanted it to have information about the next matchups for each team, so I could include some trash talk about the upcoming week.  This all required some thought - how do we pass this data into the AI in a way that it can understand?

I first decided to create a custom object in python, named `RelevantBoxScoreInfo` which is used to store the team box score info for the "home" and "away" teams, along with a reference to what matchup week we're looking at.  The team box scores are also custom objects, named `TeamBoxScoreInfo` which contains the team name, the score, the ranking, their record, and who the team is playing next week and the ranking and record of that team.  All of this information is parsed as part of the object's constructor:

```python
def __init__(self, teamStats, teamScore, teamLineup, matchupWeek):
    self.team_name = teamStats.team_name
    self.team_score = teamScore
    self.team_record = f"{teamStats.wins}-{teamStats.losses}"
    self.team_rank = teamStats.standing
    self.team_next_matchup = teamStats.schedule[matchupWeek].team_name
    self.team_next_matchup_record = f"{teamStats.schedule[matchupWeek+1].wins}-{teamStats.schedule[matchupWeek+1].losses}"
    self.team_next_matchup_rank = teamStats.schedule[matchupWeek+1].standing

    # Get player stats
    self.team_players = []
    for player in teamLineup:
        self.team_players.append(PlayerBoxScoreInfo(player))
```

"Matt, why are you passing in the score separate from the team stats?"  I'm glad you asked - it's because the score isn't a part of the team stats object.  This is one of the messy parts of how the ESPN API organized their data and why I chose not to pass the output from the ESPN API into the AI.

"Wait, so player stats come from their own object too?"  Yep - that's right - players also have their own object, named `PlayerBoxScoreInfo` which stores info about each player on the team's lineup:

```python
def __init__(self, player):
    self.player_name = player.name

    if player.lineupSlot == "BE":
        self.player_position = "BENCH"
    elif player.lineupSlot == "IR":
        self.player_position = "INJURED RESERVE"
    else:
        self.player_position = player.position
        
    self.player_predicted_points = player.projected_points
    self.player_actual_points = player.points 
```

Once again I had to get a bit creative here with the positions.  The `lineupSlot` (which comes from the API) doesn't really show the player's position.  For example - a Running Back shows up as "RB/WR/TE", but in reality they belong to the "RB" slot.  If the player was left on a team's Bench ("BN") their lineupSlot whould show as "BN" but their position would still be their normal position (e.g. "RB").  I had to do some mapping to get the player's position to show up correctly, which is why there's a small amount of mapping logic there.  I also wanted to distinguish a bench slot from an IR slot, so I added a check for that as well.

Ok cool so I have these custom Python objects, but now what?  Well the obvious answer is to JSON encode them, but the regular `json.dumps()` method in python won't work here because the `json` module doesn't know how to serialize custom objects.  Luckily, `jsonpickle` exists and can do just that, so I used that module to serialize the root `RelevantBoxScoreInfo` object into a JSON string.

Whew!  Ok, now that that's all done, we can FINALLY start to form our prompts.

### Forming the matchup prompts

For this exercise, I'm starting with the matchup prompts.  I want to generate a prompt for each matchup, so I loop through the matchups and create a `RelevantBoxScoreInfo` object for each team in the matchup.  I then serialize the object into a JSON string and pass it into the prompt.

I decided to split this prompt into two parts - a "system" prompt and a "user" prompt.  Odds are I didn't have to do that; I could have just done it as one prompt.  This system prompt contains a lot of stuff, namely examples of matchup roasts and how the data for each matchup is going to be provided.  

<details>
<summary>Click here to see the system prompt</summary>
<blockquote>
You are to respond in the voice of a sports entertainment announcer who roasts fantasy football matchups.  These should be 2-4 sentences long with one paragraph encompassing the entire matchup, and should highlight and lowlight key areas of each team's week.  Use information from the provided box score information in your analysis.  

This roast analysis will be used as part of a digest newsletter, and will be located somewhere in the middle so keep that in mind when writing it.  For all of these, assume there's an introduction and conclusion that will be written by someone else.  You're just here to roast the matchups in the middle of the newsletter.  So introductory statements like "Ladies and Gentlemen" or "Step right up!" are not necessary.  Just get right into the roasting.

Are you ready?  Are you up to this task?  Be ruthless.  we can take it.  Remember - you're roasting people.  No need to be nice.

## Information Provided:

For each team in the matchup (home and away) you'll be given the following information:
- Team Name
- Team Score
- Team Players with their individual scores (as well as what they were projected to score)

This will be represented as a JSON object with the following structure:

```json
{
  "matchupWeek": 4,
  "homeTeam": {
    "team_name": "home team name",
    "team_score": 0.0,
    "team_record": "home team record",
    "team_rank": 4,
    "team_next_matchup": "home team's next opponent",
    "team_next_matchup_record": "home team's next opponent record",
    "team_next_matchup_rank": 10,
    "team_players": [
      {
        "player_name": "Home team player",
        "player_position": "player's position",
        "player_predicted_points": 19.43,
        "player_actual_points": 23.6
      },
    ]
  },
  "awayTeam": {
    "team_name": "Away team name",
    "team_score": 0.0,
    "team_record": "Away team record",
    "team_rank": 10,
    "team_next_matchup": "Away team's next opponent",
    "team_next_matchup_record": "Away team's next opponent record",
    "team_next_matchup_rank": 4,
    "team_players": [
      {
        "player_name": "Away team player",
        "player_position": "player's position",
        "player_predicted_points": 23.6,
        "player_actual_points": 19.43
      },
    ]
  }
}
```

## Roast Examples:

> Concepts of a Team got steamrolled this week, putting up a sad 73 points while Mahomes tried to carry a squad that looked like it forgot how to play fantasy football. Tyreek Hill and Mike Evans combined for 10.7 points, which is like showing up to a cookout with a bag of ice. Meanwhile, Winner Winner Chicken Dinner was serving a five-course feast, dropping 165.58 points with Brock Purdy balling out and Dallas Goedert putting up 27 points like it's nothing. Concepts drops to 1-2, while Winner Winner stays undefeated at 3-0, living their best fantasy life.

> Jackson5 gave it a good shot, racking up 110 points with Derrick Henry dropping 30.4 and Rashee Rice adding 29.1, but E's Armadillos took the win despite a rough start from Anthony Richardson's 5.08 points. The Armadillos were carried by Saquon Barkley's monster 33.6 points and Jonathan Taylor's 26.5, leaving just enough room to edge out Jackson5. Zay Flowers and Matt Gay didn't do much, but Barkley alone made sure the Armadillos stayed strong at 2-1, while Jackson5 remains winless at 0-3 and staring down another tough week ahead.

> Winner Winner Chicken Dinner is cruising at 2-0, thanks to James Cook's explosive performance and Amon-Ra St. Brown's dazzling play, though Cooper Kupp and Dallas Goedert are cooling off like a soggy french fry. Meanwhile, Jackson5 is stuck at 0-2, despite Lamar Jackson and Derrick Henry holding the fort. They left Marvin Harrison Jr. on the bench while struggling with Ladd McConkey and Travis Kelce, making their road to victory look like a bumpy ride ahead.
</blockquote>
</details>

The user prompt is a bit simpler - it's a template string which I'll fill in dynamically:

<details>
<summary>Click here to see the user prompt</summary>
<blockquote>
NEXT MATCHUP: {home_team_name} vs. {away_team_name}.

{home_team_name} is currently {home_team_record} and ranks {home_team_rank} in the league.  Their next matchup is against {home_team_next_opponent} who is currently {home_team_next_opponent_record} and ranks {home_team_next_opponent_rank} in the league.

{away_team_name} is currently {away_team_record} and ranks {away_team_rank} in the league.  Their next matchup is against {away_team_next_opponent} who is currently {away_team_next_opponent_record} and ranks {away_team_next_opponent_rank} in the league.

Matchup Information:
{matchup_info}

Do your worst.  Remember - you're roasting people.  No need to be nice.
</blockquote>
</details>

I'm 100% egging on the AI here.  I want it to be agressive and actually roast the matchup rather than just analyze it.

This is also a template string - you can see placeholders such as `{home_team_name}` and `{matchup_info}`.  These placeholders will be replaced with the actual data when the prompt is generated.

### Forming the intro and outro prompts

The intro and outro prompts are much simpler - they're just template strings which direct the AI to write an intro and outro for the newsletter, give it the context that it is roasting a Fantasy Football league's matchups, and specifies that it should lso make some sort of joke about how this entire newsletter was AI generated.  It's produced some pretty good results so far - things like:

> WHAT'S UP, FANTASY FANS?! It’s that magical time of the week when we dive into our fantasy football showdown, guided by an AI that still believes “tight end” refers to my diet! Buckle up for some laughs, burns, and perhaps a few questionable lineup decisions that could make even your grandma cringe. Let’s kick this off!

Or as an outro:

> And that’s a wrap on another week of fantasy football antics, where some teams soared like eagles while others crash-landed harder than my AI capabilities at a karaoke night! Remember, folks, it’s all in good fun—just like me pretending to have emotions! Until next week, keep your lineups sharp and your roasts even sharper!

## Step 3 - Call the AI

RoastGPT works by calling an Azure OpenAI service instance to generate the text.  I'm using GPT-4o-mini for this since it is well-suited for tasks like creative text generation.  I thought about using the new o1 model, but I couldn't justify the extra cost of using it for this project and didn't need the analytical capabilities it can offer.  GPT-4o-mini is more than capable of doing the job.

To generate the roast messages, it's a simple call to the Azure OpenAI service.  I'm also dynamically generating the prompt messages at the same time.

```python
def generate_matchup_roast(self, relevantBoxScore):
    # JSONify the relevant box score
    jsonRelevantBoxScore = jsonpickle.encode(relevantBoxScore, unpicklable=False)

    messages = [
        {
            "role": "system",
            "content": system_matchup_prompt
        },
        {
            "role": "user",
            "content": matchup_prompt.format(
                home_team_name=relevantBoxScore.homeTeam.team_name,
                home_team_record=relevantBoxScore.homeTeam.team_record,
                home_team_rank=relevantBoxScore.homeTeam.team_rank,
                home_team_next_opponent = relevantBoxScore.homeTeam.team_next_matchup,
                home_team_next_opponent_record = relevantBoxScore.homeTeam.team_next_matchup_record,
                home_team_next_opponent_rank = relevantBoxScore.homeTeam.team_next_matchup_rank,
                away_team_name=relevantBoxScore.awayTeam.team_name,
                away_team_record=relevantBoxScore.awayTeam.team_record,
                away_team_rank=relevantBoxScore.awayTeam.team_rank,
                away_team_next_opponent = relevantBoxScore.awayTeam.team_next_matchup,
                away_team_next_opponent_record = relevantBoxScore.awayTeam.team_next_matchup_record,
                away_team_next_opponent_rank = relevantBoxScore.awayTeam.team_next_matchup_rank,
                matchup_info=jsonRelevantBoxScore)
        }
    ]

    # Ship it to the AI and let them roast it...
    roast = self.gptClient.chat.completions.create(
        messages=messages,
        model = self.gptModel
    )

    return roast.choices[0].message.content
```

This is a pretty simple method - fill in the placeholders in the prompt, call the Azure OpenAI service, and parse the response for the message content.  Rinse and repeat for the rest of the matchups, and you have a full newsletter recap generated by an AI.

The intro and outro prompts are generated in a similar way, but they're a bit simpler since they don't require any dynamic data to be passed in.  I'm not going to show that, you'll just have to take my word for it.

## The Results

All in all, the AI did a pretty good job....  Here's how it handled this week's matchup (I didn't put the full newsletter here, just a snippet):

> WHAT IS UP FANTASY FANS?!?! It’s that glorious time of the week when our lineups get dissected by an AI with all the emotional depth of a crumb! Buckle up, because I’ve got hot takes hotter than a dragon's breath and roasts sharper than a rogue's dagger. LET'S DO THIS!!!
>
> Jackson5 took to the gridiron this week like a dream team, racking up an impressive 130.74 points, led by a Derrick Henry bull rush that had defenders wishing they had opted for a career in accounting. Marvin Harrison Jr. and Jordan Mason added solid performances, while Lamar Jackson reminded us just how crucial he is with a slick 23.64-point outing. Meanwhile, Vanessa’s Spirited Team stumbled through their fantasy matchup like they were running a three-legged race in a banana suit, accumulating a mere 84.4 points. Breece Hall flopped harder than a new sitcom on a Friday night, posting just 3.8, and only Michael Pittman Jr. seemed to know what a football was with his respectable 17.3. The Spirited Team is now 0-4 and searching for answers like a kid lost in a game store—good luck with that! Keep those spirits up, because the only thing guaranteed this season is heartbreak.
>
> And that’s a wrap on this week’s fantasy fiascos! Remember, folks, fantasy football is a lot like me—sometimes it sparks joy, and other times it leaves you wondering how someone so artificial could roast you so well! Now go set your lineups and may your players perform better than a robot trying to write a love song!

Ok, it could use some work on the outro.  But overall, I'm pretty happy with how it turned out, especially for a project that took only a couple of hours to put together.  

## Conclusion

This was a fun project - I got to combine my love of Fantasy Football trash-talking with my interest in using AI to generate creative content.  All in all, I'm pretty happy with how it turned out, and based on the feedback from the folks in my fantasy leagues they're enjoying it too.  This AI tool does really nothing ground-breaking or super useful, it was just a fun project that I wanted to do, and one that I wanted to share.  It's now getting so easy to interface with these AI tools that you can do some pretty cool stuff with them in just a couple of hours, and my goal here was to inspire you all to do the same.

Oh, and in case you were wondering - the entire idea for this project, the name "RoastGPT," and the prompt ideas were all inspired and assisted by an AI.  Yep, that's right, with a little bit of prompting and some creative thinking, an AI was able to inspire me to write a script to interface with an AI, then the AI helped me write the prompts, then the AI helped me write the blog post.