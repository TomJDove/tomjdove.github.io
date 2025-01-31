---
layout: post
title: "AFL and SQL"
date: 2023-07-28
categories: 
permalink: '/sideprojects/afl-sql'
---

Here, I use SQL to explore [this Kaggle dataset](https://www.kaggle.com/datasets/stoney71/aflstats), which gives basic match and player statistics for each AFL match from 2012 to 2021.

## Creating the database

To start, I needed to create the database:

```sql
CREATE TABLE game (
    gameID VARCHAR(255),
    year YEAR,
    round VARCHAR(25),
    date DATE,
    venue VARCHAR(255),
    startTime TIME,
    attendance INT,
    homeTeam VARCHAR(255),
    homeTeamScore INT,
    awayTeam VARCHAR(255),
    awayTeamScore INT,
    rainfall DECIMAL(4,1),
    PRIMARY KEY (gameID)
);

CREATE TABLE player (
    playerID INT,
    displayName VARCHAR(255),
    height SMALLINT,
    weight SMALLINT,
    dob DATE,
    position VARCHAR(255),
    origin VARCHAR(255),
    PRIMARY KEY (playerID)
);

CREATE TABLE stat (
    gameID VARCHAR(255),
    team VARCHAR(255),
    playerID INT,
    gameNumber SMALLINT,
    disposals SMALLINT,
    kicks SMALLINT,
    marks SMALLINT,
    handballs SMALLINT,
    goals SMALLINT,
    behinds SMALLINT,
    hitouts SMALLINT,
    tackles SMALLINT,
    rebounds SMALLINT,
    inside50s SMALLINT,
    clearances SMALLINT,
    clangers SMALLINT,
    frees SMALLINT,
    frees_against SMALLINT,
    brownlow_votes SMALLINT,
    contested_possessions SMALLINT,
    uncontested_possessions SMALLINT,
    contested_marks SMALLINT,
    marks_inside_50 SMALLINT,
    one_percenters SMALLINT,
    bounces SMALLINT,
    goal_assists SMALLINT,
    percent_played SMALLINT,
    FOREIGN KEY (gameID) REFERENCES game(gameID),
    FOREIGN KEY (playerID) REFERENCES player(playerID)
);

```

I then used a LOAD DATA query to populate the tables with the AFL data, which are in csv files.

## Some basic queries

```sql
/* Number of games each year */
SELECT YEAR(match_date) AS season,
	COUNT(gameID) AS match_count
FROM game
GROUP BY season
ORDER BY season;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table1.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
Here, we see the effect of COVID in 2020.
The missing game in 2015 stands out: a match between Adelaide and Geelong was cancelled due to the death of Phil Walsh, Adelaide's coach.
</p>


```sql
/* Best goal kickers since 2012 */
SELECT player.playerID, displayName, SUM(goals) AS total_goals
FROM player INNER JOIN stat
	ON player.playerID = stat.playerID
GROUP BY playerID
ORDER BY total_goals DESC
LIMIT 10;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table2.png"
  style=""
  >
</figure>
</center>


```sql
/* Most goals kicked in a single season */
SELECT player.playerID, 
    displayName, 
    YEAR(match_date) AS season, 
    SUM(goals) AS total_goals
FROM player 
	INNER JOIN stat
		ON player.playerID = stat.playerID
	INNER JOIN game
		ON stat.gameID = game.gameID
GROUP BY season, playerID
ORDER BY total_goals DESC
LIMIT 10;
```

<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table3.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
In recent years, no player has managed to crack the fabled 100 goals in a season.
Only 28 players have achieved this, the most recent being Lance Franklin in 2008.
</p>

```sql
/* Average points scored per game each year */
SELECT 
    YEAR(match_date) AS season,
    ROUND(AVG(homeTeamScore + awayTeamScore), 1) AS avg_total_points
FROM game
GROUP BY season;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table4.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
To get stats on individual teams, it will be useful to separate the match data so that each entry corresponds to a single team:
</p>


```sql
CREATE TEMPORARY TABLE team_matches AS
SELECT gameID, match_date, homeTeam AS team, homeTeamScore AS score, 
	awayTeam AS opponent, awayTeamScore AS opponent_score
FROM game
UNION
SELECT gameID, match_date, awayTeam AS team, awayTeamScore AS score, 
	homeTeam AS opponent, homeTeamScore AS opponent_score
FROM game;
```

```sql
/* Highest number of points scored by a team in a single match */
SELECT match_date, team, score
FROM team_matches
ORDER BY score DESC
LIMIT 10;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table5.png"
  style=""
  >
</figure>
</center>

## A grand prediction

<p align='justify'>
Imaging it's the 25th of September, 2021 and you're sitting with your family about to watch the Dees take on the Dogs in the grand final.
To impress your family with your knowledge of sports, you want to predict the winner.
You don't actually have much knowledge of sports, but you can look at the data.
</p>

Start by seeing where each team sits on the ladder:


```sql
/* The 2021 ladder up to the grand final */
SELECT team, 
    SUM(2*(score > opponent_score) + (score = opponent_score)) AS total_points
FROM team_matches
WHERE YEAR(match_date) = 2021 AND match_date < '2021-09-25'
GROUP BY team
ORDER BY total_points DESC;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table6.png"
  style=""
  >
</figure>
</center>

Unsurprisingly, two grand final teams hold the top two positions on the ladder.

<p align='justify'>
A naive prediction at this point would be that Melbourne wins, but we can consider more salient statistics.
Let's look at some basic stats for the teams' performance over the season:
</p>

```sql
/* Various performance metrics for the season so far */
WITH team_stats AS (
	SELECT gameID, team, 
        SUM(contested_possessions) AS total_contested_possessions,
        SUM(marks_inside_50) AS total_marks_inside_50
	FROM stat
    WHERE team = 'Melbourne' OR team = 'Western Bulldogs'
    GROUP BY gameID, team
)
SELECT 
    team_stats.team,
    ROUND(AVG(score - opponent_score), 2) AS avg_point_diff,
    ROUND(AVG((score > opponent_score)*2 + (score = opponent_score)), 2) 
        AS avg_performance,
    ROUND(AVG(total_contested_possessions), 2) AS avg_contested_possessions,
    ROUND(AVG(total_marks_inside_50), 2) AS avg_marks_inside_50
FROM team_stats INNER JOIN team_matches
	ON team_stats.gameID = team_matches.gameID 
		AND team_stats.team = team_matches.team
WHERE YEAR(match_date) = 2021 AND match_date < '2021-09-25'
GROUP BY team
ORDER BY team;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table7.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
We already know from the ladder that Melbourne has performed better. 
They have also done better in terms of contested possessions.
The Bulldogs have a slightly better points differential and a few more marks inside 50, but it's not a great difference.
</p>

<p align='justify'>
How about each teams' performance in recent games?
</p>

```sql
/* The total number of points scored by and against each team in the last 5 games 
   together with the average performance in these games */
WITH numbered_games AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY team ORDER BY match_date DESC
        ) AS game_number
    FROM team_matches
    WHERE match_date < '2021-09-25'
)
SELECT 
    team, 
    SUM(score) AS points_for_last5, 
    SUM(opponent_score) AS points_against_last5,
    SUM(score - opponent_score) AS points_diff_last5,
    ROUND(AVG( 
        2*(score > opponent_score) + (score = opponent_score) 
    ), 1) AS avg_performance_last5
FROM numbered_games
WHERE game_number <= 5 AND team IN ('Melbourne', 'Western Bulldogs')
GROUP BY team;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table8.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
So, despite Western Bulldogs having a slightly better average point differential over the season, Melbourne has a significantly better point differential over the previous five games.
This is likely more significant because most of these five games were played in the finals, so they could've been playing stronger teams.
</p>

<p align='justify'>
Meters gained and the inside 50 mark differential tend to provide a strong link to success, at least according to <a href = 'https://www.statsinsider.com.au/news/afl-stats-that-matter-what-are-the-real-performance-indicators'>statsinsider.com</a>.
We don't have data on meters gained, but we can calculate the inside 50 mark differential:
</p>

```sql
/* Get marks inside 50 differential */
WITH team_marks AS (
    /* Group individual player mark stats into team level stats */
    SELECT gameID, team, SUM(marks_inside_50) AS team_marks_inside_50
    FROM stat
    GROUP BY gameID, team
),
team_marks_diff AS (
    /* Obtain inside 50 differentials */
    SELECT tg.gameID, 
        tg.team, 
        tm1.team_marks_inside_50 - tm2.team_marks_inside_50
            AS marks_inside_50_diff
    FROM team_matches tg
        INNER JOIN team_marks tm1
            ON tg.gameID = tm1.gameID 
                AND tg.team = tm1.team
        INNER JOIN team_marks tm2
            ON tg.gameID = tm2.gameID 
                AND tg.opponent= tm2.team
    WHERE YEAR(match_date) = 2021 AND match_date < '2021-09-25'
)
SELECT team, SUM(marks_inside_50_diff) AS sum_marks_inside_50_diff 
FROM team_marks_diff
GROUP BY team
ORDER BY sum_marks_inside_50_diff DESC;
```
<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/table9.png"
  style=""
  >
</figure>
</center>

<p align='justify'>
Indeed, if we graph the marks inside 50 differential with the performance (which is calculated as +2 points for a win and +1 for a draw), we see a correlation:
</p>

<center>
<figure>
  <img
  src="https://tomjdove.github.io/TomJDove/assets/afl_sql/chart1.png"
  style="width:75%"
  >
  <figcaption>
    Marks inside 50 differential correlates with performance.
  </figcaption>
</figure>
</center>

So, based on our findings, it seems wise to tip Melbourne to win (and indeed they did).
