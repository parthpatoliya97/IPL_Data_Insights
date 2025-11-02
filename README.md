# IPL 2021-2023 Data Insights

## Quick Refresh to Basic Cricket Metrics :-
### 🏏Batting Average

#### Meaning: On average, how many runs a batter scores before getting out.
- Batting Average = Total Runs Scored / Total Times Out
- Higher average = consistent scorer.

### ⚡Strike Rate (SR)

#### Meaning: How fast a batter scores.
- Strike Rate = (Total Runs / Total Balls Faced) × 100
- Higher SR = aggressive and impactful batter.

### 🎯 Boundary Percentage

#### Meaning: What percentage of total runs came from fours & sixes.
- Boundary % = (Runs from 4s & 6s / Total Runs) × 100
- Shows how attacking a batter is.

### ⛔ Dot Ball Percentage by Batter

#### Meaning: Percentage of balls where batter scored 0 runs.
- Dot Ball % = (Dot Balls / Total Balls Faced) × 100
- Lower dot-ball % means better strike rotation.

### 🏹 Bowling Average

#### Meaning: How many runs a bowler concedes per wicket taken.
- Bowling Average = Runs Conceded / Wickets Taken
- Lower average = better wicket-taking bowler.

### 💰 Economy Rate

#### Meaning: Runs conceded per over by a bowler.
- Economy Rate = Runs Conceded / Overs Bowled
- Shows how economical the bowler is. Lower is better.

### 🟧 Orange Cap

#### Meaning: Batter with the most total runs in a season.

### 🟪 Purple Cap

#### Meaning: Bowler with most wickets in a season.

---------------------------------------------------------------------------------------------------------------------------------------------

### 1.) Top 10 Batsman with Most Runs over last 3 Years
```sql

select
    batsmanName,
    count(*) as total_matches,
    sum(runs) as total_runs
from fact_bating_summary
group by batsmanName
order by total_runs desc
limit 10;
```
![top 10 batsman](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_batsman_with_most_runs_last_3_years_with%20_total_matches.png?raw=true)

### 2.) Top 10 Batsman with Most Runs in each season
```sql
DELIMITER $$

CREATE PROCEDURE get_top_batsmen_by_season(IN in_season INT)
BEGIN
    WITH season_stats AS (
        SELECT
            YEAR(STR_TO_DATE(dms.matchDate, '%b %d,%Y')) AS IPL_Season,
            fbs.batsmanName,
            SUM(fbs.runs) AS total_runs,
            COUNT(*) AS total_innings
        FROM fact_bating_summary fbs
        JOIN dim_match_summary dms
            ON fbs.match_id = dms.match_id
        GROUP BY IPL_Season, fbs.batsmanName
        HAVING SUM(fbs.balls) >= 60
    ),
    season_average AS (
        SELECT
            IPL_Season,
            batsmanName,
            total_runs,
            total_innings,
            DENSE_RANK() OVER (
                PARTITION BY IPL_Season
                ORDER BY total_runs DESC
            ) AS ranking
        FROM season_stats
    )
    SELECT 
        IPL_Season,
        batsmanName,
        total_innings,
        total_runs
    FROM season_average
    WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
    ORDER BY IPL_Season, total_runs DESC;
END$$

DELIMITER ;

```
![2021 top run getter](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_10_batsman_based_on_runs.png?raw=true)
![2022 top run getter](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_batsman_based_on_runs.png?raw=true)
![2023 top run getter](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_batsman_based_on_runs.png?raw=true)

### 3.) Top 10 Batsman with highest Batting Average in last 3 years(Total Matches>30)
```sql
WITH season_stats AS (
    SELECT
        fbs.batsmanName,
        SUM(fbs.runs) AS total_runs,
        SUM(fbs.balls) AS balls_faced,
        COUNT(*) AS total_innings,
        COUNT(CASE WHEN fbs.`out/not_out` = 'out' THEN 1 END) AS total_outs
    FROM fact_bating_summary fbs
    JOIN dim_match_summary dms
        ON fbs.match_id = dms.match_id
    GROUP BY fbs.batsmanName
    HAVING SUM(fbs.balls) >= 60 and count(*)>30
),
season_average AS (
    SELECT
        batsmanName,
        total_runs,
        balls_faced,
        total_innings,
        total_outs,
        ROUND(
            CASE WHEN total_outs = 0 THEN total_runs ELSE total_runs / total_outs END,
            2
        ) AS batting_average,
        DENSE_RANK() OVER (
            ORDER BY
                CASE WHEN total_outs = 0 THEN total_runs ELSE total_runs / total_outs END DESC
        ) AS ranking
    FROM season_stats
)
SELECT batsmanName,total_innings,total_runs,batting_average
FROM season_average
WHERE ranking <= 10
ORDER BY batting_average DESC;
```
![top 10 batting average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_batsman_batting_average_30_matches.png?raw=true)


### 4.) Top 10 Batsman with highest Batting Average in each season
```sql
DELIMITER $$

CREATE PROCEDURE get_top_batting_average_by_season(IN in_season INT)
BEGIN
    WITH season_stats AS (
        SELECT
            YEAR(STR_TO_DATE(dms.matchDate, '%b %d,%Y')) AS IPL_Season,
            fbs.batsmanName,
            SUM(fbs.runs) AS total_runs,
            SUM(fbs.balls) AS balls_faced,
            COUNT(*) AS total_innings,
            COUNT(CASE WHEN fbs.`out/not_out` = 'out' THEN 1 END) AS total_outs
        FROM fact_bating_summary fbs
        JOIN dim_match_summary dms
            ON fbs.match_id = dms.match_id
        GROUP BY IPL_Season, fbs.batsmanName
        HAVING SUM(fbs.balls) >= 60
    ),
    season_average AS (
        SELECT
            IPL_Season,
            batsmanName,
            total_runs,
            balls_faced,
            total_innings,
            total_outs,
            ROUND(
                CASE WHEN total_outs = 0 THEN total_runs 
                     ELSE total_runs / total_outs END,
            2) AS batting_average,
            DENSE_RANK() OVER (
                PARTITION BY IPL_Season
                ORDER BY 
                    CASE WHEN total_outs = 0 THEN total_runs 
                         ELSE total_runs / total_outs END DESC
            ) AS ranking
        FROM season_stats
    )
    SELECT 
        IPL_Season,
        batsmanName,
        total_innings,
        total_runs,
        batting_average
    FROM season_average
    WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
    ORDER BY IPL_Season, batting_average DESC;
END$$

DELIMITER ;

```
![2021 top 10 batting average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_TOP_10bastam_based_on_batting_average.png?raw=true)
![2022 top 10 batting average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_batsman_on_batting_average.png?raw=true)
![2023](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_batsman_based_on_batting_average.png?raw=true)


### 5.) Top 10 batsman with highest batting strike rate in last 3 years(Total Matches>30)
```sql
WITH season_stats AS (
    SELECT
        fbs.batsmanName,
        COUNT(*) AS total_innings,
        SUM(fbs.runs) AS total_runs,
        SUM(fbs.balls) AS balls_faced
    FROM fact_bating_summary fbs
    JOIN dim_match_summary dms
        ON fbs.match_id = dms.match_id
    GROUP BY fbs.batsmanName
    HAVING SUM(fbs.balls) >= 60 and count(*)>30
),
season_SR AS (
    SELECT
        batsmanName,
        total_innings,
        total_runs,
        balls_faced,
        ROUND(
            CASE
                WHEN balls_faced = 0 THEN 0
                ELSE (total_runs / balls_faced) * 100
            END, 2
        ) AS batting_strike_rate,
        DENSE_RANK() OVER (
            ORDER BY
                CASE
                    WHEN balls_faced = 0 THEN 0
                    ELSE (total_runs / balls_faced) * 100
                END DESC
        ) AS ranking
    FROM season_stats
)
SELECT batsmanName,total_innings,total_runs,batting_strike_rate
FROM season_SR
WHERE ranking <= 10
ORDER BY  batting_strike_rate DESC;
```
![top 10 batting strike rate over 30 matches](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_batsman_strike_rate_30_matches.png?raw=true)

### 6.) Top 10 batsman with highest batting strike rate in each season
```sql
DELIMITER $$

CREATE PROCEDURE GetTopStrikeRateBySeason(IN season_year INT)
BEGIN
    WITH season_stats AS (
        SELECT
            YEAR(STR_TO_DATE(dms.matchDate, '%b %d,%Y')) AS IPL_Season,
            fbs.batsmanName,
            COUNT(*) AS total_innings,
            SUM(fbs.runs) AS total_runs,
            SUM(fbs.balls) AS balls_faced
        FROM fact_bating_summary fbs
        JOIN dim_match_summary dms
            ON fbs.match_id = dms.match_id
        GROUP BY IPL_Season, fbs.batsmanName
        HAVING SUM(fbs.balls) >= 60
    ),
    season_SR AS (
        SELECT
            IPL_Season,
            batsmanName,
            total_innings,
            total_runs,
            balls_faced,
            ROUND(
                CASE
                    WHEN balls_faced = 0 THEN 0
                    ELSE (total_runs / balls_faced) * 100
                END, 2
            ) AS batting_strike_rate,
            DENSE_RANK() OVER (
                PARTITION BY IPL_Season
                ORDER BY
                    CASE
                        WHEN balls_faced = 0 THEN 0
                        ELSE (total_runs / balls_faced) * 100
                    END DESC
            ) AS ranking
        FROM season_stats
    )
    SELECT IPL_Season,batsmanName,total_innings,total_runs,batting_strike_rate
    FROM season_SR
    WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
    ORDER BY batting_strike_rate DESC;
END $$

DELIMITER ;

```
![2021 top 10 strike rate](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_10_batsman_based_on_batting_strike_rate.png?raw=true)
![2022 top 10 strike rate](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_batsman_based_on_batting_strike_rate.png?raw=true)
![2023 top 10 strike rate](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_batsman_based_on_batting_strike_rate.png?raw=true)


### 7.) Top 10 Wicket Takers over last 3 years
```sql
select
    bowlerName,
    count(*) as total_matches,
    sum(wickets) as total_wickets
from fact_bowling_summary
group by bowlerName
order by total_wickets desc
limit 10;
```
![top 10 wicket takers](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_highest_wicket_takers_in_last_3_years.png?raw=true)


### 8.) Top 10 Wicket Takers in each season
```sql
DELIMITER $$

CREATE PROCEDURE top_wicket_takers(IN season_year INT)
BEGIN
WITH season_stats AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate,'%b %d,%Y')) AS IPL_Season,
        fbs.bowlerName,
        SUM(fbs.wickets) AS total_wickets
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms ON fbs.match_id=dms.match_id
    GROUP BY IPL_Season, fbs.bowlerName
    HAVING SUM(fbs.overs*6) >= 60
),
season_rank AS (
    SELECT *,
           DENSE_RANK() OVER (PARTITION BY IPL_Season ORDER BY total_wickets DESC) AS ranking
    FROM season_stats
)
SELECT IPL_Season,bowlerName,total_wickets
FROM season_rank
WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
ORDER BY total_wickets DESC;
END$$

DELIMITER ;

```
![2021 top 10 bowler](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_10_bowlers.png?raw=true)
![2022 top 10 bowler](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_bowlers.png?raw=true)
![2023 top 10 bowler](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_bowlers.png?raw=true)

### 11.) top 10 best bowling average bowlers in last 3 years(Total Matches>30)
```sql
WITH season_stats AS (
    SELECT
        fbs.bowlerName,
        count(*) as total_mathces,
        SUM(fbs.wickets) AS total_wickets,
        SUM(fbs.runs) AS total_runs_conceded,
        SUM(fbs.overs * 6) AS balls_bowled
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms
        ON fbs.match_id = dms.match_id
    GROUP BY fbs.bowlerName
    HAVING SUM(fbs.overs * 6) >= 60 and count(*)>30
),
season_average AS (
    SELECT
        bowlerName,
        total_mathces,
        total_wickets,
        total_runs_conceded,
        ROUND(
            CASE WHEN total_wickets = 0 THEN NULL
                 ELSE total_runs_conceded / total_wickets END,
            2
        ) AS bowling_average,
        DENSE_RANK() OVER (
            ORDER BY
                CASE WHEN total_wickets = 0 THEN 999
                     ELSE total_runs_conceded / total_wickets END
        ) AS ranking
    FROM season_stats
)
SELECT
    bowlerName,
    total_mathces,
    total_wickets,
    bowling_average
FROM season_average
WHERE ranking <= 10
ORDER BY bowling_average;

```
![top 10 bowling average last 3 years](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_bowling_average_with_30_matches.png?raw=true)

### 12.) Top 10 best bowling average bowlers in each season
```sql
DELIMITER $$

CREATE PROCEDURE sp_best_bowling_avg(IN season_year INT)
BEGIN
WITH season_stats AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate,'%b %d,%Y')) AS IPL_Season,
        fbs.bowlerName,
        SUM(fbs.wickets) AS total_wickets,
        SUM(fbs.runs) AS total_runs_conceded,
        SUM(fbs.overs*6) AS balls_bowled
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms ON fbs.match_id=dms.match_id
    GROUP BY IPL_Season, fbs.bowlerName
    HAVING SUM(fbs.overs*6) >= 60
),
avg_rank AS (
    SELECT *,
           ROUND(
               CASE WHEN total_wickets = 0 THEN NULL
                    ELSE total_runs_conceded/total_wickets END, 2) AS bowling_average,
           DENSE_RANK() OVER (
               PARTITION BY IPL_Season
               ORDER BY CASE WHEN total_wickets = 0 THEN 999
                             ELSE total_runs_conceded/total_wickets END
           ) AS ranking
    FROM season_stats
)
SELECT IPL_Season,bowlerName,total_wickets,bowling_average
FROM avg_rank
WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
ORDER BY bowling_average ASC;
END$$

DELIMITER ;

```
![2021 top 10 bowling average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_10_best_bowling_average_bowlers.png?raw=true)
![2022 top 10 bowling average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_best_bowling_Average_bowlers.png?raw=true)
![2023 top 10 bowling average](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_best_bowling_average_bowlers.png?raw=true)

### 13.) Top 10 Economical Bowlers over last 3 years(Total Matches>30) 
```sql
WITH season_stats AS (
    SELECT
        fbs.bowlerName,
        count(*) as total_matches,
        SUM(fbs.wickets) AS total_wickets,
        SUM(fbs.runs) AS total_runs_conceded,
        SUM(fbs.overs) AS total_overs,
        SUM(fbs.overs * 6) AS balls_bowled
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms
        ON fbs.match_id = dms.match_id
    GROUP BY fbs.bowlerName
    HAVING SUM(fbs.overs * 6) >= 60 and count(*)>30
),
season_economy AS (
    SELECT
        bowlerName,
        total_matches,
        total_wickets,
        total_runs_conceded,
        total_overs,
        ROUND(
            CASE
                WHEN total_overs = 0 THEN NULL
                ELSE total_runs_conceded / total_overs
            END, 2
        ) AS economy_rate,
        DENSE_RANK() OVER (
            ORDER BY
                CASE
                    WHEN total_overs = 0 THEN 999
                    ELSE total_runs_conceded / total_overs
                END
        ) AS ranking
    FROM season_stats
)
SELECT
    bowlerName,
    total_matches,
    total_wickets,
    economy_rate
FROM season_economy
WHERE ranking <= 10
ORDER BY economy_rate;

```
![top 10 economy over 3 years](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_economical_bowlers_with_30_matches.png?raw=true)

### 14.) Top 10 Economical Bowlers in each seasons
```sql
DELIMITER $$

CREATE PROCEDURE sp_best_economy(IN season_year INT)
BEGIN
WITH season_stats AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate,'%b %d,%Y')) AS IPL_Season,
        fbs.bowlerName,
        COUNT(*) AS total_matches,
        SUM(fbs.wickets) AS total_wickets,
        SUM(fbs.runs) AS total_runs_conceded,
        SUM(fbs.overs) AS total_overs
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms ON fbs.match_id=dms.match_id
    GROUP BY IPL_Season, fbs.bowlerName
    HAVING SUM(fbs.overs*6) >= 60
),
eco_rank AS (
    SELECT *,
           ROUND(total_runs_conceded/total_overs,2) AS economy_rate,
           DENSE_RANK() OVER (PARTITION BY IPL_Season ORDER BY total_runs_conceded/total_overs) AS ranking
    FROM season_stats
)
SELECT IPL_Season,bowlerName,total_matches,total_wickets,economy_rate
FROM eco_rank
WHERE ranking <= 10
      AND (in_season IS NULL OR IPL_Season = in_season)
ORDER BY economy_rate ASC;
END$$

DELIMITER ;

```
![2021 top 10 economy](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_best_economical_bowlers.png?raw=true)
![2022 top 10 economy](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_best_economical_bowlers.png?raw=true)
![2023 top 10 economy](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_best_economical_bowlers.png?raw=true)


### 15.) Top 10 Players with highest boundary percentage rate over last 3 years(Total Matches>30)
```sql
WITH season_stats AS (
    SELECT
        fbs.batsmanName,
        count(*) as total_matches,
        SUM(fbs.runs) AS total_runs,
        SUM(fbs.`4s`*4) AS runs_from_4s,
        SUM(fbs.`6s`*6) AS runs_from_6s
    FROM fact_bating_summary fbs
    JOIN dim_match_summary dms
         ON dms.match_id = fbs.match_id
    GROUP BY fbs.batsmanName
    having sum(fbs.balls)>=60 and count(*)>30
),
boundary_percent AS (
    SELECT
        batsmanName,
        total_matches,
        total_runs,
        runs_from_4s,
        runs_from_6s,
        ROUND((runs_from_4s + runs_from_6s) / total_runs*100, 2) AS boundary_percentage,
        DENSE_RANK() OVER (
            ORDER BY ROUND((runs_from_4s + runs_from_6s) / total_runs*100, 2) DESC
        ) AS ranking
    FROM season_stats
)
SELECT batsmanName,total_matches,total_runs,boundary_percentage
FROM boundary_percent
WHERE ranking <= 10
ORDER BY boundary_percentage DESC;
```
![top 10 boundary percentage](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_10_batsman_with_boundary_percent_last_3_years.png?raw=true)

### 16.) Top 10 Players with highest boundary percentage rate in each season
```sql
DELIMITER $$

CREATE PROCEDURE sp_boundary_percent(IN season_year INT)
BEGIN
WITH season_stats AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate,'%b %d,%Y')) AS IPL_Season,
        fbs.batsmanName,
        SUM(fbs.runs) AS total_runs,
        SUM(fbs.`4s`*4) AS runs_from_4s,
        SUM(fbs.`6s`*6) AS runs_from_6s
    FROM fact_bating_summary fbs
    JOIN dim_match_summary dms ON fbs.match_id=dms.match_id
    GROUP BY IPL_Season,fbs.batsmanName
    HAVING SUM(fbs.balls)>=60
),
bdy_rank AS (
    SELECT *,
           ROUND((runs_from_4s + runs_from_6s)/total_runs*100,2) AS boundary_percentage,
           DENSE_RANK() OVER (PARTITION BY IPL_Season 
                              ORDER BY (runs_from_4s + runs_from_6s)/total_runs DESC) AS ranking
    FROM season_stats
)
SELECT IPL_Season,batsmanName,total_runs,boundary_percentage
FROM bdy_rank
WHERE  ranking <= 5 
      AND (in_season IS NULL OR IPL_Season = in_season)
ORDER BY boundary_percentage DESC;
END$$

DELIMITER ;

```
![2021 top 10 boundary percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_10_batsman_with_boundary_percent.png?raw=true)
![2022 top 10 boundary percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_10_batsman_with_boundary_percent.png?raw=true)
![2023 top 10 boundary percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_batsman_with_boundary_percent.png?raw=true)


### 17.) Top 10 Bowlers with most dot ball% over last 3 years(Total Matches>30)
```sql
with stats as (
    select
       fbs.bowlerName,
       sum(fbs.wickets) as total_wickets,
       ROUND(sum(fbs.overs*6),0) as total_balls,
       sum(fbs.`0s`) as dot_balls
from fact_bowling_summary fbs
join dim_match_summary dms
on fbs.match_id=dms.match_id
group by fbs.bowlerName
having sum(fbs.overs*6)>=60 and count(*)>30
),
dot_balls_percent as (
    select
           bowlerName,
           total_wickets,
           total_balls,
           dot_balls,
           round(dot_balls/total_balls*100,2) as dot_ball_percentage,
           dense_rank() over (order by round(dot_balls/total_balls*100,2) desc) as ranking
    from stats
)
select bowlerName,total_wickets,total_balls,dot_balls,dot_ball_percentage
from dot_balls_percent
where ranking<=10
order by dot_ball_percentage desc;

```
![top 10 dot ball percent over 3 years](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top%2010%20dot%20ball%20percentage%20over%203%20years.png?raw=true)

### 18.) Top 10 Bowlers with most dot ball% in each season
```sql
DELIMITER $$

CREATE PROCEDURE sp_dot_ball_percent(IN season_year INT)
BEGIN
WITH stats AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate,'%b %d,%Y')) AS IPL_Season,
        fbs.bowlerName,
        SUM(fbs.wickets) AS total_wickets,
        SUM(fbs.overs*6) AS total_balls,
        SUM(fbs.`0s`) AS dot_balls
    FROM fact_bowling_summary fbs
    JOIN dim_match_summary dms ON fbs.match_id=dms.match_id
    GROUP BY IPL_Season,fbs.bowlerName
    HAVING SUM(fbs.overs*6)>=60
),
dot_rank AS (
    SELECT *,
           ROUND(dot_balls/total_balls*100,2) AS dot_ball_percentage,
           DENSE_RANK() OVER (PARTITION BY IPL_Season ORDER BY dot_balls/total_balls DESC) AS ranking
    FROM stats
)
SELECT IPL_Season,bowlerName,total_wickets,total_balls,dot_balls,dot_ball_percentage
FROM dot_rank
WHERE ranking <= 5 
      AND (in_season IS NULL OR IPL_Season = in_season)
ORDER BY dot_ball_percentage DESC;
END$$

DELIMITER ;

```
![2021 top 5 dot ball percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2021_top_5_dot_ball_percent_bowlers.png?raw=true)
![2022 top 5 dot ball percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2022_top_5_dot_ball%25.png?raw=true)
![2023 top 5 dot ball percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/2023_top_10_balls_percent.png?raw=true)


### 19.) Top 4 teams based on past 3 years winning %.
```sql
  WITH all_matches AS (
    SELECT
        team1 AS team,
        CASE WHEN team1 = winner THEN 1 ELSE 0 END AS win_flag
    FROM dim_match_summary

    UNION ALL

    SELECT
        team2 AS team,
        CASE WHEN team2 = winner THEN 1 ELSE 0 END AS win_flag
    FROM dim_match_summary
),
team_stats AS (
    SELECT
        team,
        COUNT(*) AS matches_played,
        SUM(win_flag) AS wins,
        ROUND((SUM(win_flag) / COUNT(*)) * 100, 2) AS win_percentage
    FROM all_matches
    GROUP BY team
)
SELECT *
FROM team_stats
ORDER BY win_percentage DESC
LIMIT 4;
```
![winning percent](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_4_team_by_winning_%25_last_3_years.png?raw=true)

### 20.) Top 5 All-Rounders

- All-rounder Score = (Bowling Avg * Economy Rate) / Batting Strike Rate
- All-rounder Score (lower is better bowling, higher is better batting)
- Lower score = better all-rounder.

```sql
WITH batting AS (
    SELECT
        dp.name AS Player,
        count(*) AS total_matches,
        SUM(fb.runs) AS Total_Runs,
        SUM(fb.balls) AS Total_Balls,
        ROUND((SUM(fb.runs) / NULLIF(SUM(fb.balls),0)) * 100, 2) AS Batting_SR
    FROM fact_bating_summary fb
    JOIN dim_players dp
        ON fb.batsmanName = dp.name
    WHERE dp.playingRole = 'Allrounder'
    GROUP BY dp.name
    HAVING SUM(fb.balls) >= 60 and count(*)>15
),

bowling AS (
    SELECT
        dp.name AS Player,
        SUM(fb.wickets) AS Total_Wickets,
        SUM(fb.runs) AS Runs_Conceded,
        SUM(fb.overs) AS Overs,
        SUM(fb.overs * 6) AS Balls_Bowled,
        ROUND(SUM(fb.runs) / NULLIF(SUM(fb.wickets),0), 2) AS Bowling_Avg,
        ROUND(SUM(fb.runs) / NULLIF(SUM(fb.overs),0), 2) AS Economy
    FROM fact_bowling_summary fb
    JOIN dim_players dp
        ON fb.bowlerName = dp.name
    WHERE dp.playingRole = 'Allrounder'
    GROUP BY dp.name
    HAVING SUM(fb.overs * 6) >= 60
       AND SUM(fb.wickets) > 0 AND count(*)>15
),

combined AS (
    SELECT
        b.Player,
        b.total_matches,
        b.Total_Runs,
        b.Batting_SR,
        bw.Total_Wickets,
        bw.Bowling_Avg,
        bw.Economy,
        ROUND((bw.Bowling_Avg * bw.Economy) / b.Batting_SR, 3) AS Allrounder_Score
    FROM batting b
    JOIN bowling bw
        ON b.Player = bw.Player
    WHERE b.Batting_SR IS NOT NULL
      AND bw.Bowling_Avg IS NOT NULL
)

SELECT *
FROM combined
ORDER BY Allrounder_Score ASC
LIMIT 5;
```
![top all rounders](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/top_5_all_rounders.png?raw=true)

### 21.) Orange Cap Winners in each season
```sql
-- orange cap
WITH cte AS (
    SELECT
        YEAR(STR_TO_DATE(dms.matchDate, '%b %d,%Y')) AS IPL_Season,
        fbs.batsmanName AS Batsman_Name,
        SUM(fbs.runs) AS Total_Runs
    FROM fact_bating_summary fbs
    JOIN dim_match_summary dms
        ON fbs.match_id = dms.match_id
    GROUP BY
        YEAR(STR_TO_DATE(dms.matchDate, '%b %d,%Y')),
        fbs.batsmanName
),

orange_cap AS (
    SELECT
        IPL_Season,
        Batsman_Name,
        Total_Runs,
        DENSE_RANK() OVER (PARTITION BY IPL_Season ORDER BY Total_Runs DESC) AS ranking
    FROM cte
)

SELECT IPL_Season,Batsman_Name,Total_Runs
FROM orange_cap
WHERE ranking = 1 and IPL_Season is not null
ORDER BY IPL_Season
```
![orange cap](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/orange_cap.png?raw=true)

### 22.) Purple Cap Winners in each season
```sql
with cte as(
select year(str_to_date(dms.matchDate,'%b %d,%Y')) as IPL_Season,
       fbs.bowlerName as Bowler_Name,
       sum(fbs.wickets) as Total_wickets
from fact_bowling_summary fbs
join dim_match_summary dms
on fbs.match_id=dms.match_id
group by year(str_to_date(dms.matchDate,'%b %d,%Y')),bowlerName),
purple_cap as(
select IPL_Season,Bowler_Name,Total_wickets,dense_rank() over (partition by IPL_Season order by Total_wickets desc) as ranking
from cte)
select IPL_Season,Bowler_Name,Total_wickets
from  purple_cap
where ranking=1 and IPL_Season is not null
order by IPL_Season
```
![purple cap](https://github.com/parthpatoliya97/IPL_Data_Insights/blob/main/Query_Results_images/purple_cap.png?raw=true)


