---
title: "Lab 3: Mayoral Mystery"
toc: false
---

<!-- Import Data -->
```js
const nyc = await FileAttachment("data/nyc.json").json();
const results = await FileAttachment("data/election_results.csv").csv({ typed: true });
const survey = await FileAttachment("data/survey_responses.csv").csv({ typed: true });
const events = await FileAttachment("data/campaign_events.csv").csv({ typed: true });

// // Note: you don't have to keep this, but some helpful data exposure to see what we've loaded. 
// // NYC geoJSON data
// display(nyc)
// // Campaign data (first 10 objects)
// display(results.slice(0,10))
// display(survey.slice(0,10))
// display(events.slice(0,10))
```

```js //prepare district boundaries
// The nyc file is saved in data as a topoJSON instead of a geoJSON. Thats primarily for size reasons -- it saves us 3MB of data. For Plot to render it, we have to convert it back to its geoJSON feature collection. 
const districts = topojson.feature(nyc, nyc.objects.districts)
// display(districts)
```

```js
// Simple rendering of the NYC districts topoJSON
// Plot.plot({
//   // this projection is already zoomed into NYC
//   projection: {
//     domain: districts,
//     type: "mercator",
//   },
//   marks: [
//     Plot.geo(districts),
//   ]
// })
```

<style>
body {
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  color: #334155;
  line-height: 1.6;
}

.intro-box {
  background: linear-gradient(135deg, #1e40af 0%, #3b82f6 100%);
  color: white !important; /* Forces all text to be white */
  padding: 32px;
  border-radius: 16px;
  margin-bottom: 40px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
}
.intro-box h1, .intro-box h2, .intro-box h3 {
  color: #ffffff !important;
  margin-top: 0;
}
.intro-box ol, .intro-box ul {
  color: #e2e8f0;
}

.section-box {
  background: white;
  padding: 30px;
  border-radius: 12px;
  margin-bottom: 40px;
  border-top: 4px solid #3b82f6; 
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
}
h1 { font-size: 2rem; font-weight: 800; letter-spacing: -0.025em; }
h2 { font-size: 1.25rem; color: #64748b; font-weight: 400; }

.result-text {
  background-color: #f8fafc;
  padding: 20px;
  border-radius: 8px;
  margin-top: 20px;
  border-left: 4px solid #cbd5e1;
}
.result-text h1 {
  font-size: 1.1rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #475569;
  margin-bottom: 10px;
}
</style>

# Welcome to the Mayoral Mystery (Lab #3)
## By Sidrat Habib
<br>

<div class="intro-box">

# Context

A candidate entered the 2024 NYC mayoral race with strong support in several districts but narrowly lost the election overall. Using election results, survey responses, and campaign event data, this dashboard explores where the campaign succeeded, where it struggled, and what strategic changes may help in a future election. During this analysis, we attempted to: 

1. experiment working with maps and geospatial data
2. build data exploration skillsets with ambiguity
3. iterate with visualizations to discover data insights
4. hone the data → visualization workflow

<br>

### The analysis focuses on four major questions:

1. Which districts supported the candidate the most?
2. How did income level relate to election performance?
3. Did campaign outreach improve turnout and vote share?
4. Which campaign issues connected best with voters?

</div>

```js //prepare election data 
const resultsWithShare = results.map(d => ({
  ...d,
  vote_share: d.votes_candidate / (d.votes_candidate + d.votes_opponent)
}));

const voteShareMap = new Map(
  resultsWithShare.map(d => [String(d.boro_cd), d.vote_share])
);

function districtCode(feature) {
  return String(
    feature.properties.BoroCD ??
    feature.properties.boro_cd
  );
}
```

<br>
<br>

<div class="section-box">

# Candidate Vote Share Across NYC

```js
Plot.plot({
  width: 850,
  height: 700,
  projection: {
    domain: districts,
    type: "mercator"
  },
  color: {
    scheme: "blues",
    percent: true,
    label: "Candidate Vote Share",
    legend: true
  },
  marks: [
    Plot.geo(districts, {
      fill: d => voteShareMap.get(districtCode(d)),
      stroke: "white",
      strokeWidth: 0.7,
      title: d => {
        const code = districtCode(d);
        const share = voteShareMap.get(code);
        return `District ${code}\nVote Share: ${(share * 100).toFixed(1)}%`;
      },
      tip: true
    })
  ]
})
```

  <div class="result-text">

# Results

The map shows that support for the candidate was not evenly distributed across New York City. Stronger performance appeared mainly in parts of the Bronx and Brooklyn, with many districts reaching over <strong>60%</strong> vote share. In contrast, weaker performance appeared in Staten Island and wealthier Manhattan districts, where support often dipped below <strong>40%</strong>.

The geographic divide suggests that the campaign message connected more strongly with working-class districts than wealthier neighborhoods. This pattern becomes even clearer when comparing election performance by income category.
  </div> 
</div>

<br>
<br>

<div class="section-box">

# Vote Share by Income Category 

```js
const incomeSummary = d3.rollups(
  resultsWithShare,
  v => d3.mean(v, d => d.vote_share) * 100,
  d => d.income_category
).map(([income, average_share]) => ({
  income,
  average_share
}));

display(
  Plot.plot({
    width: 700,
    height: 450,
    marginLeft: 60,
    marginBottom: 50,
    x: {
      label: "Income Category"
    },
    y: {
      label: "Average Vote Share (%)",
      domain: [0, 70]
    },
    marks: [
      Plot.barY(incomeSummary, {
        x: "income",
        y: "average_share",
        fill: "steelblue",
        title: d => `${d.income}\n${d.average_share.toFixed(1)}%`
      }),
      Plot.text(incomeSummary, {
        x: "income",
        y: "average_share",
        text: d => `${d.average_share.toFixed(1)}%`,
        dy: -10
      }),
      Plot.ruleY([0])
    ]
  })
)
```
  <div class="result-text">

# Results

Income level was strongly connected to candidate performance. Low-income districts had the highest average vote share at <strong>58.4%</strong>, while high-income districts had the weakest support at just <strong>28.8%</strong>.

This suggests the campaign’s strongest coalition came from lower-income communities where issues like affordable housing and public transportation may have been especially important. Middle-income districts were much more competitive at <strong>49.1%</strong>, meaning they could become the decisive swing districts in a future campaign.

  </div> 
</div>

<br>
<br>

<div class="section-box">

# GOTV Effort vs Candidate Vote Share

```js
Plot.plot({
  width: 800,
  height: 500,
  marginLeft: 60,
  marginBottom: 50,
  x: {
    label: "Doors Knocked"
  },
  y: {
    label: "Candidate Vote Share",
    tickFormat: "%"
  },
  color: {
    legend: true,
    label: "Income Category"
  },
  marks: [
    Plot.dot(resultsWithShare, {
      x: "gotv_doors_knocked",
      y: "vote_share",
      fill: "income_category",
      r: 5,
      opacity: 0.8,
      title: d => `District ${d.boro_cd}\nDoors Knocked: ${d.gotv_doors_knocked}\nVote Share: ${(d.vote_share * 100).toFixed(1)}%`,
      tip: true
    })
  ]
})
```

  <div class="result-text">

# Results

The scatterplot shows a positive relationship between doors knocked and candidate vote share. Low-income districts received the largest GOTV effort, with many seeing over <strong>7,000 doors</strong> knocked, resulting in consistently high performance.

In contrast, many middle-income districts received much less outreach, often fewer than <strong>1,000</strong> doors, even though they remained highly competitive (ranging from <strong>40% to 58%</strong> vote share). This suggests the campaign’s ground game was effective where applied, but resources were not distributed efficiently; increasing outreach in those middle-income clusters could significantly improve overall results.

  </div> 
</div>

<br>
<br>

<div class="section-box">

# Voter Alignment on Key Issues

```js
const issueData = survey.flatMap(d => [
  { issue: "Housing", score: d.affordable_housing_alignment },
  { issue: "Transit", score: d.public_transit_alignment },
  { issue: "Childcare", score: d.childcare_support_alignment },
  { issue: "Small Business Tax", score: d.small_business_tax_alignment },
  { issue: "Police Reform", score: d.police_reform_alignment }
]);

const issueSummary = d3.rollups(
  issueData,
  v => d3.mean(v, d => d.score),
  d => d.issue
).map(([issue, average]) => ({
  issue,
  average
})).sort((a, b) => d3.descending(a.average, b.average));

display(
  Plot.plot({
    width: 750,
    height: 450,
    marginLeft: 70,
    marginBottom: 60,
    y: {
      label: "Average Alignment Score",
      domain: [0, 5]
    },
    x: {
      label: "Issue"
    },
    marks: [
      Plot.barY(issueSummary, {
        x: "issue",
        y: "average",
        fill: "darkorange",
        title: d => `${d.issue}\n${d.average.toFixed(2)}`
      }),
      Plot.text(issueSummary, {
        x: "issue",
        y: "average",
        text: d => d.average.toFixed(1),
        dy: -10
      }),
      Plot.ruleY([0])
    ]
  })
)
```
  <div class="result-text">
  
# Results

Affordable Housing and Public Transit received the strongest alignment scores from survey respondents, both averaging <strong>3.9</strong> out of <strong>5.0</strong>. Childcare support also performed relatively well with a <strong>3.6</strong> score.

Police Reform received the lowest alignment score by a massive margin, averaging only <strong>1.4</strong>, suggesting voters were significantly out of step with the candidate’s position on that issue. Overall, the survey data suggests the campaign’s strongest messaging pillars are housing and transit, while the police reform platform acted as a major deterrent for voters.

  </div> 
</div>

<br>
<br>

<div class="section-box">

# Final Recommendation

The data suggests the campaign was competitive but struggled with resource allocation and issue messaging.

Three major patterns stand out:

1. The candidate performed best in low-income districts and struggled most in high-income districts.
2. GOTV outreach appeared effective, especially in districts with higher door-knocking activity.
3. Housing and transit policies were popular with voters, while police reform created weaker alignment.

## For a future campaign, the strongest strategy would likely involve:

1. Increasing outreach in competitive middle-income districts
2. Continuing to focus on housing and transit messaging
3. Reconsidering how police reform policies are communicated to voters
4. Spending more candidate time in swing districts rather than heavily unfavorable districts

Overall, the election results suggest the race was winnable with more targeted outreach and stronger strategic focus.

</div>