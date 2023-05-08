Disparities in Response Rates: An Investigation
================
Brooke Moss, Lily Jiang, Anna Letcher Hartman, Bethany Costello
2023-05-07

- [Background](#background)
  - [The Dataset](#the-dataset)
  - [Methodology](#methodology)
  - [Our Question](#our-question)
- [Investigation](#investigation)
  - [Questions v. Attorneys and Hours](#questions-v-attorneys-and-hours)
  - [Outside Factors](#outside-factors)
  - [Web Accessibility](#web-accessibility)
- [Recommendations](#recommendations)
- [References](#references)

## Background

### The Dataset

During DataFest 2023, we were given datasets representing the American
Bar Association’s Free Legal Answers program (ABA FLA). The Free Legal
Answers program is an online platform where the ABA provides pro bono
(free of charge) legal services to most US states. Qualifying people
(based on income thresholds) can submit questions to be researched and
answered by volunteer lawyers \[DataFest Challenge Description\].

The data we received represents a full history of the program from
August 2016 to January 2022, including client information, attorney
information, correspondence details, and conversation text. The dataset
appears to be completely comprehensive as it contains all questions
asked through the online FLA program. However, because some of the
variables (such as the hours logged by each attorney) are reliant on
self-reporting measures, parts of the dataset are not particularly
trustworthy. These variables contain variation that is impossible to
account for.

### Methodology

Using this data, our team investigated disparities in attorney response
rates across states within the program. In this program, qualifying
users can submit legal questions, which are then “accepted” by an
attorney. The attorney researches the question and communicates back and
forth with the user to answer.

We determined that if an attorney “accepted” a question (taking it into
their purview) we would count that as a question that has been responded
to. However, if no attorney accepted the question, we would count that
as a non-response.

In order to count responses, we used the `TakenByAttorneyUno` variable
from the `questions.csv` dataset. This variable represents the unique
identifier of the attorney that accepted the question. If no attorney
accepted it, then the column value was `NULL` and we counted it as a
non-response. This is surely an under-counting of questions that were
unresolved, as this column only indicates whether or not an attorney
responded to the original question. This fails to account for
non-response cases in which a conversation occurs between client and
attorney before an attorney stops responding or questions left
unresolved.

### Our Question

``` r
# Create a helper tibble connecting FIPS codes with counties
states_and_fips <- fips_codes %>%
  mutate(
    fips = paste0(state_code, county_code)
  ) %>%
  select(state, county, fips) %>%
  mutate(county = str_remove_all(county, " County"))

# Count the number of questions unanswered from each county in the US
questions_by_county <- questions %>%
  left_join(select(clients, -Id), by = c("StateAbbr", "AskedByClientUno" = "ClientUno")) %>%
  group_by(County, StateAbbr) %>%
  mutate(
    is_null = 1 * (TakenByAttorneyUno == "NULL"),
  ) %>%
  count(is_null) %>%
  pivot_wider(
    names_from = "is_null",
    names_prefix = "null",
    values_from = "n"
  ) %>%
  mutate(
    null_percent = null1 / (null0 + null1)
  ) %>%
  drop_na() %>%
  left_join(states_and_fips, by = c("County" = "county", "StateAbbr" = "state")) %>%
  arrange(desc(null_percent)) %>%
  drop_na()

# Plot the null response rates by county for Texas and Florida
plot_usmap(include = c("FL", "TX"), regions = "counties", data = questions_by_county, values = "null_percent", color = NA) +
  scale_fill_continuous(
    low = "white", high = "red", name = "Nonresponse rate", label = scales::comma
  ) +
  labs(title = "Nonresponse rates for Texas and Florida") +
  theme(legend.position = "bottom")
```

![](Report_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

``` r
questions %>% 
  filter(
    StateAbbr == c('TX', 'FL')
  ) %>%
  ggplot() +
  geom_histogram(aes(x = AskedOnUtc, alpha = 0.2), bins = 72) +
  geom_histogram(
    data = . %>%
      filter(TakenByAttorneyUno != "NULL"),
    aes(x = AskedOnUtc, fill = StateAbbr),
    bins = 72
  ) +
  facet_wrap(facets =  vars(StateAbbr)) 
```

    ## Warning: There was 1 warning in `filter()`.
    ## ℹ In argument: `StateAbbr == c("TX", "FL")`.
    ## Caused by warning in `StateAbbr == c("TX", "FL")`:
    ## ! longer object length is not a multiple of shorter object length

![](Report_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

When digging into response rates, we noticed a large discrepancy between
the two states with the largest number of questions: Texas and Florida.
These states appeared to have begun the FLA program at a similar time
and had a similar number of active attorneys, number of attorney hours,
and questions asked. However, the response rate in Florida was 87%, as
opposed to 47% in Texas. This led to our research question: **what
accounts for the diﬀerences in response rates among states?**

## Investigation

We narrowed down the states to investigate by limiting our time scale to
Jan 1st, 2020 to January 24, 2022. We then grabbed the states with
roughly over 100 questions per month, or 2400 questions in total over
two years. These were, in descending order of response rates: Tennessee,
Wisconsin, Florida, North Carolina, Missouri, Virginia, New York, South
Carolina, Illinois, Massachusetts, Indiana, Texas, Oklahoma, Arizona,
and Georgia.

### Questions v. Attorneys and Hours

![](Report_files/figure-gfm/hours-perattorney-1.png)<!-- -->

``` r
df_responses %>% 
  left_join(attorney_time_state, by = "StateAbbr") %>%
  mutate(casePer = total_questions/num_attorney) %>% 
  ggplot(aes(
    x = casePer,
    y = response_rate)
  ) +
  geom_point(aes(color = StateAbbr)) +
  geom_text_repel(
    mapping = aes(label = StateAbbr, color = StateAbbr)
  ) +
  ylim(0, 1) +
  xlab("Total Number of Questions Per Active Attorney") +
  ylab("Response Rate (Questions / Questions Taken by Attorney)") +
  ggtitle("Response Rate vs. Number of Questions per Active Attorney by State") +
  theme(legend.position="none")
```

![](Report_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

To answer this, we calculated the number of unique attorneys per state
that logged any time on the website within our 2 year time scale. We
also calculated the total number of hours per state. This allowed us to
calculate the average time spent per attorney by state. We compared this
against the response rate and created a scatter plot. We also plotted
the average number of questions per active attorney over the response
rate. This allowed us to better explain the disparities found within
Georgia, Oklahoma, and Arizona. All had a low number of hours logged by
the attorney, as well as had a high number of questions per attorney.
This is especially true in Arizona, with almost 160 questions per
attorney. It is clear that demand far outweighs supply in these states.
Texas was not as straightforward. They had a similar number of questions
per attorney as Florida, but with a much lower response rate. Even more
strangely, they had an exorbitant amount of hours spent per question. We
looked to see if there was a higher number of average posts per
question, but Texas didn’t seem to be any diﬀerent than other states.

### Outside Factors

We turned to outside research to make sense of response variation. We
researched pro bono policies for states. Four of the states (FL, NY, IN,
IL) required attorneys to report the number of pro bono hours per year.
Six states (TN, NC, VA, TX, AZ, GA) allowed attorneys to voluntarily
report the number of pro bono hours per year. The remaining 5 (WI, MO,
SC, MA, OK) had no policy around reporting. There were other interesting
externalities that could be impacting response rates. New York state
requires 50 hours of pro bono work for admission to the Bar. Florida,
Tennessee, and Wisconsin have more of a culture of doing pro bono work
than other states. A 2018 report by the ABA found that Tennessee ranked
2nd in a survey of attorneys in 24 states, with 67% reporting having
done some pro bono work. This is similar to the 57% found in a 2016 pro
bono survey in Wisconsin and the 52% found in a 2006 study of Florida
attorneys. This contrasts greatly with the 38.4% of Texas attorneys that
completed pro bono work found in a 2019 report by the Texas State Bar.
Without access to the surveys that these reports drew upon, it was hard
for us to draw conclusions about the ways in which the culture of pro
bono work diﬀers across states and could aﬀect response rates.

### Web Accessibility

![](Report_files/figure-gfm/web-accessibility-1.png)<!-- -->

We also investigated how accessible each state’s ABA FLA was from each
state’s bar association website. For some states, such as Tennessee and
Florida, we found the link to Free Legal Answers featured on the list of
pro bono opportunities for members. For others, such as Oklahoma and
Georgia, the link was hidden among a myriad of in person and remote pro
bono opportunities. We found it diﬃcult to create a metric for this
disparity. We attempted a number of clicks but this was diﬃcult to
standardize among the vastly diﬀerent website layouts. We ﬁnally decided
we would create a boolean variable for whether or not the FLA link was
featured and easily noticeable on the pro bono opportunity website for
lawyers who were not searching outright for the website. Then, we
plotted it. This appeared to have an eﬀect on response rates.

## Recommendations

We would recommend that the ABA ask state bar associations to list the
FLA link among pro bono opportunities for members. We also encourage the
ABA to continue their work with the Baylor University School of Law and
the Stanford Legal Design Lab creating auto emails for users waiting for
an attorney to take their question. These emails containing answers to
frequently asked questions could make a real diﬀerence in states with
low response rates. We also urge the ABA to provide more surveys to
Texas attorneys to make sense of the low response rate and large amount
of time spent per question. We want to ensure that this trend does not
continue in other states as this program expands.

<!--# Do we want to elaborate?? -->

## References

- <https://www.tncourts.gov/sites/default/files/docs/atj_2016_pro_bono_report.pdf>
- <https://www.wisbar.org/formembers/probono/Documents/Pro%20Bono%20Survey%20Report_WI%202016.pdf>
- <https://www.floridabar.org/public/probono/probono002/>
- <https://www.americanbar.org/content/dam/aba/administrative/probono_public_service/ls_pb_supporting_justice_iv_final.authcheckdam.pdf>
- <https://www.abalegalprofile.com/pro-bono.php#anchor4>
- <https://www.americanbar.org/groups/probono_public_service/policy/bar_pre_admission_pro_bono/>
- <https://www.americanbar.org/groups/probono_public_service/policy/arguments/>
- <https://www.texasbar.com/Content/NavigationMenu/LawyersGivingBack/LegalAccessDivision/ProBonoSurvey.pdf>
