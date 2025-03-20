SELECT *
FROM bank_customers_staging2;

# This approach avoids unintended decimal division and works across different SQL engines
# In some SQL databases conversions/total * 100 might round down to 0 if the result is less than 1
# before multiplying by 100
#1 CONVERSION RATE
SELECT 
    COUNT(CASE WHEN y = 'yes' THEN 1 END) AS conversion_total,
    COUNT(*) AS total_contacts,
    (COUNT(CASE WHEN y = 'yes' THEN 1 END)* 100) / COUNT(*) AS conversion_rate
FROM
    bank_customers_staging2;

# Demographic segmentation as percentage of those who converted
# Gives breakdown of age, num of conversions by age and the % 
# of the total converted population that represents as a %
# used a subquery as the denominator in the calc
SELECT age, COUNT(*) as num_of_conversions,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE y = 'yes'), 2)
AS percent_of_total_conversions
FROM bank_customers_staging2
WHERE y = 'yes'
GROUP BY age
ORDER BY percent_of_total_conversions DESC;

#2 age_group_perc_conversions
SELECT 
CASE
	WHEN age BETWEEN 17 AND 19 THEN '17-19'
    WHEN age BETWEEN 20 AND 29 THEN '20-29'
    WHEN age BETWEEN 30 AND 39 THEN '30-39'
    WHEN age BETWEEN 40 AND 49 THEN '40-49'
    WHEN age BETWEEN 50 AND 59 THEN '50-59'
    WHEN age BETWEEN 60 AND 69 THEN '60-69'
    WHEN age BETWEEN 70 AND 79 THEN '70-79'
    WHEN age BETWEEN 80 AND 89 THEN '80-89'
    WHEN age > 89 THEN '90+'
END AS age_group, 
COUNT(*) as num_of_conversions,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE y = 'yes'), 2)
AS percent_of_total_conversions
FROM bank_customers_staging2
WHERE y = 'yes'
GROUP BY age_group
ORDER BY percent_of_total_conversions DESC;

#3 job_conversions
# Now to segment all conversions by job
# repeating the same process as above
# Demographic segmentation as percentage of those who converted
SELECT job, COUNT(*) as num_of_conversions,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE y = 'yes'), 2)
AS percent_of_total_conversions
FROM bank_customers_staging2
WHERE y = 'yes'
GROUP BY job
ORDER BY percent_of_total_conversions DESC;

#4 Now to segment all conversions by education
# repeating the same process as above
# Demographic segmentation as percentage of those who converted
SELECT education, COUNT(*) as num_of_conversions,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE y = 'yes'), 2)
AS percent_of_total_conversions
FROM bank_customers_staging2
WHERE y = 'yes'
GROUP BY education
ORDER BY percent_of_total_conversions DESC;

#5 Now to segment all conversions by marital
# repeating the same process as above
# Demographic segmentation as percentage of those who converted
SELECT marital, COUNT(*) as num_of_conversions,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE y = 'yes'), 2)
AS percent_of_total_conversions
FROM bank_customers_staging2
WHERE y = 'yes'
GROUP BY marital
ORDER BY percent_of_total_conversions DESC;

# Now to start combining demograpihics to see what dominates conversions

WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	CASE
		WHEN age BETWEEN 17 AND 19 THEN '17-19'
		WHEN age BETWEEN 20 AND 29 THEN '20-29'
		WHEN age BETWEEN 30 AND 39 THEN '30-39'
		WHEN age BETWEEN 40 AND 49 THEN '40-49'
		WHEN age BETWEEN 50 AND 59 THEN '50-59'
		WHEN age BETWEEN 60 AND 69 THEN '60-69'
		WHEN age BETWEEN 70 AND 79 THEN '70-79'
		WHEN age BETWEEN 80 AND 89 THEN '80-89'
		WHEN age > 89 THEN '90+'
	END AS age_group, job, education, marital,
    COUNT(*) as num_of_conversions, # Have to use MAX() here as tc.total_conversions is a single number
    # as any column in the SELECT that is not inside an aggregate function (like COUNT(), SUM(), AVG())
	# must be explicitly included in the GROUP BY clause. But we cant put in group by as is single number!
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1 # Can use CROSS JOIN instead but does the same
# Have to use this as no shared column between cte and main table!
# 1=1 evaluates to true and as cte holds only 1 number
WHERE y = 'yes' 
AND age BETWEEN 20 AND 59
  AND job IN ('admin', 'technician', 'blue-collar', 'retired', 'management')
  AND education IN ('university.degree', 'high.school', 'professional.course', 'basic.9y')
  AND marital IN ('married', 'single')
GROUP BY age_group, job, education, marital
ORDER BY percent_total_conversions DESC;

#6 Too granular so panned out tried age and jobs
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	CASE
		WHEN age BETWEEN 17 AND 19 THEN '17-19'
		WHEN age BETWEEN 20 AND 29 THEN '20-29'
		WHEN age BETWEEN 30 AND 39 THEN '30-39'
		WHEN age BETWEEN 40 AND 49 THEN '40-49'
		WHEN age BETWEEN 50 AND 59 THEN '50-59'
		WHEN age BETWEEN 60 AND 69 THEN '60-69'
		WHEN age BETWEEN 70 AND 79 THEN '70-79'
		WHEN age BETWEEN 80 AND 89 THEN '80-89'
		WHEN age > 89 THEN '90+'
	END AS age_group, job,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND age BETWEEN 20 AND 59
AND job IN ('admin', 'technician', 'blue-collar', 'retired', 'management')
GROUP BY age_group, job
ORDER BY percent_total_conversions DESC;

#7 age and education ****
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	CASE
		WHEN age BETWEEN 17 AND 19 THEN '17-19'
		WHEN age BETWEEN 20 AND 29 THEN '20-29'
		WHEN age BETWEEN 30 AND 39 THEN '30-39'
		WHEN age BETWEEN 40 AND 49 THEN '40-49'
		WHEN age BETWEEN 50 AND 59 THEN '50-59'
		WHEN age BETWEEN 60 AND 69 THEN '60-69'
		WHEN age BETWEEN 70 AND 79 THEN '70-79'
		WHEN age BETWEEN 80 AND 89 THEN '80-89'
		WHEN age > 89 THEN '90+'
	END AS age_group,education,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND age BETWEEN 20 AND 59
AND education IN ('university.degree', 'high.school', 'professional.course', 'basic.9y')
GROUP BY age_group,education
ORDER BY percent_total_conversions DESC;

#8 tried age and marital ****
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	CASE
		WHEN age BETWEEN 17 AND 19 THEN '17-19'
		WHEN age BETWEEN 20 AND 29 THEN '20-29'
		WHEN age BETWEEN 30 AND 39 THEN '30-39'
		WHEN age BETWEEN 40 AND 49 THEN '40-49'
		WHEN age BETWEEN 50 AND 59 THEN '50-59'
		WHEN age BETWEEN 60 AND 69 THEN '60-69'
		WHEN age BETWEEN 70 AND 79 THEN '70-79'
		WHEN age BETWEEN 80 AND 89 THEN '80-89'
		WHEN age > 89 THEN '90+'
	END AS age_group, marital,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND age BETWEEN 20 AND 59
AND marital IN ('married', 'single')
GROUP BY age_group, marital
ORDER BY percent_total_conversions DESC;

#9 job and education
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	job, education,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND job IN ('admin', 'technician', 'blue-collar', 'retired', 'management')
AND education IN ('university.degree', 'high.school', 'professional.course', 'basic.9y')
GROUP BY job, education
ORDER BY percent_total_conversions DESC;

#10 Job and Marital
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	job, marital,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND job IN ('admin', 'technician', 'blue-collar', 'retired', 'management')
AND marital IN ('married', 'single')
GROUP BY job, marital
ORDER BY percent_total_conversions DESC;

#11 Education and marital
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	education, marital,
    COUNT(*) as num_of_conversions,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
AND education IN ('university.degree', 'high.school', 'professional.course', 'basic.9y')
AND marital IN ('married', 'single')
GROUP BY education, marital
ORDER BY percent_total_conversions DESC;

## Question 2 ####

SELECT *
FROM bank_customers_staging2;

#1 Overall contacts by contact method
SELECT contact, COUNT(*) AS contact_type_num,
ROUND((COUNT(*)/(SELECT COUNT(*) FROM bank_customers_staging2)*100) ,2) AS percent_of_dataset
FROM bank_customers_staging2
WHERE contact = 'cellular'
OR contact = 'telephone'
GROUP BY contact;

#2 Contact method from converted cohort
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
GROUP BY contact
ORDER BY percent_total_conversions DESC;

#3 Contact method from failed cohort
WITH total_fails_cte AS(
	SELECT COUNT(*) AS total_fails
	FROM bank_customers_staging2
    WHERE y = 'no'
)
SELECT
	contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_fails), 2) AS percent_total_fails
FROM bank_customers_staging2 
JOIN total_fails_cte AS tc
ON 1=1
WHERE y = 'no' 
GROUP BY contact
ORDER BY percent_total_fails DESC;

#4 Now to calculate conversion rate by contact method
# To discover if cell is more successful or just more vol
SELECT contact, COUNT(*) AS converted_contacts,
(SELECT COUNT(*) FROM bank_customers_staging2 WHERE contact = 'cellular') AS total_cell_contacts,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE contact = 'cellular') ,2) AS conversion_rate
FROM bank_customers_staging2
WHERE contact = 'cellular'
AND y = 'yes'
GROUP BY contact;

#5 Now telephone
SELECT contact, COUNT(*) AS converted_contacts,
(SELECT COUNT(*) FROM bank_customers_staging2 WHERE contact = 'telephone') AS total_phone_contacts,
ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM bank_customers_staging2 WHERE contact = 'telephone') ,2) AS conversion_rate
FROM bank_customers_staging2
WHERE contact = 'telephone'
AND y = 'yes'
GROUP BY contact;

# Now to check demographics against contact methods to see if patterns emerge
#6 Contact method from converted cohort starting with age range
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes'
)
SELECT
	CASE
		WHEN age BETWEEN 17 AND 19 THEN '17-19'
		WHEN age BETWEEN 20 AND 29 THEN '20-29'
		WHEN age BETWEEN 30 AND 39 THEN '30-39'
		WHEN age BETWEEN 40 AND 49 THEN '40-49'
		WHEN age BETWEEN 50 AND 59 THEN '50-59'
		WHEN age BETWEEN 60 AND 69 THEN '60-69'
		WHEN age BETWEEN 70 AND 79 THEN '70-79'
		WHEN age BETWEEN 80 AND 89 THEN '80-89'
		WHEN age > 89 THEN '90+'
	END AS age_group,
	contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes' 
GROUP BY age_group, contact
ORDER BY percent_total_conversions DESC;

#7 Contact method from converted cohort with job
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes' 
)
SELECT
	job, contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes'
AND job IN ('student','admin','blue-collar','entrepreneur',
'services','technician','housemaid','management','self-employed',
'unemployed','unknown','retired')
GROUP BY job, contact
ORDER BY percent_total_conversions DESC;

#8 Contact method from converted cohort with education
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes' 
)
SELECT
	education, contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes'
AND education IN ('unknown', 'basic.9y', 'basic.4y', 'basic.6y', 'high.school',
'professional.course', 'university.degree', 'illiterate')
GROUP BY education, contact
ORDER BY percent_total_conversions DESC;


#9 Contact method from converted cohort with marital
WITH total_conversions_cte AS(
	SELECT COUNT(*) AS total_conversions
	FROM bank_customers_staging2
    WHERE y = 'yes' 
)
SELECT
	marital, contact,
    COUNT(*) as contact_type_num,
	ROUND((COUNT(*) * 100) / MAX(tc.total_conversions), 2) AS percent_total_conversions
FROM bank_customers_staging2 
JOIN total_conversions_cte AS tc
ON 1=1
WHERE y = 'yes'
AND marital IN ('married', 'single', 'divorced')
GROUP BY marital, contact
ORDER BY percent_total_conversions DESC;

# Are we contacting customers too frequently, or not frequently enough?
#10 So here I wanted the headline numbers and included y column as context
SELECT previous AS num_calls_prev_campaign,
poutcome AS result_prev_campaigns,
COUNT(previous) AS totals, y
FROM bank_customers_staging2
WHERE poutcome = 'success'
GROUP BY previous , poutcome , y
ORDER BY y DESC , poutcome , previous;

#11 
SELECT previous AS num_calls_prev_campaign, 
poutcome AS result_prev_campaigns, 
COUNT(previous) AS totals, y
FROM bank_customers_staging2
WHERE poutcome = 'failure'
GROUP BY previous, poutcome, y
ORDER BY y desc, poutcome, previous;

select *
from bank_customers_staging2;

# Q3 Campaign Timing: 
# Are there specific months or days of the week where our campaign
# is more effective? 
# Can we optimize our campaign schedule to maximize conversions?
# Starting at the macro level I wanted to look at successes 
# and failures in isolation with months and days

#1 So here by success / month
SELECT month_num, poutcome, y, COUNT(*) AS num_yes,
ROUND((COUNT(*)) / (SELECT COUNT(*) 
FROM bank_customers_staging2 
WHERE poutcome = 'success' AND y = 'yes') * 100 ,2)
AS percent_total_success
FROM bank_customers_staging2
WHERE poutcome = 'success'
AND y = 'yes'
GROUP BY month_num
ORDER BY COUNT(*) DESC;

#2 success/day
SELECT weekday_num, poutcome, y, COUNT(*) AS num_yes,
ROUND((COUNT(*)) / (SELECT COUNT(*) 
FROM bank_customers_staging2 
WHERE poutcome = 'success' AND y = 'yes') * 100 ,2)
AS percent_total_success
FROM bank_customers_staging2
WHERE poutcome = 'success'
AND y = 'yes'
GROUP BY weekday_num
ORDER BY COUNT(*) DESC;

-- End of Analysis
