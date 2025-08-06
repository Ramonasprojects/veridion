# veridion
Approach		
1. match input_company name to:		
company name (attemps to match to legal name or commercial name possible, but particularly for legal name there are some blanks in the dataset)		
2. match countries		
this poses some challenges, as there are companies where name matches and group company seems to be same, but country does not match; and there are companies for which there are several correct matches with the group, but only one/some would indicate the subsidiary in the same country		
maybe country importance to be assessed		
3. email (one of all fields) and website possible manual check for failed attempt to match, or find way to inlcude in initial matching condition sets		
4. if we know expected industries we could perhaps check against the list of industries at the end (possibly see if arts/universities and such are wrong matches)		
5. as enrichment possibilities: industry, revenue, employee count, country as primary fields for client to do analysis; secondary are the year founded, nacis, sics, naces, and descriptions with business tags, and year founded, and technolgies		
6.possible create at/updated at could show deprecated data and help to exclude rows		
		
Challenge		
1. file could not be uploaded to MySQL workbench. Attempts to save as UTF8 from Excel, UTF 8 without BOM from VC Code did not fix issue. Manually removed description columns and reimported; data appeared but only first 9 rows. Turned to file to JSON and all data was imported but column structure broke and empty rows were created. Tried VBA I found to clean line breaks inside cells and special/foreign characters		
2. since entire file could not be used I split it and only took names and countries to run code, the after SQL retrived the matches (1 row per id) I used Power Query to match agains the excel file so we can have all coulmns		
3. how to match? This was the initial concern, and I found no foolproof approach		
challenges: numbers in company names appearing as text, words being glue together		
possible ways: use basic functions like trim, lower, replace to remove white spaces, deal with capitalization, replace dots, dashes etc - but it would not improve the outcome by a lot		
Reasearched what I could do; created Levenshtein distance function using Chat GPT (apprach the issue by the number of character edits needed to turn one string into another); but did not return very good outcomes		
Since issses with the file (possibly from manual line breaks, quotes, etc) I also tried some addons to be able to fuzzy macth directly in Google Sheets but they did not work		
Another aproach in SQL to split each name by words, check how many match, and also take into account country. I removed the instances where countries were different; and dealt with them separately, by then trying the soundex function to match by how similar the words sound. Again, had to use Power Query to match this subset of data to the initial file		
So, isses that still persisted at the end:		
matches done by preserving country were at times problematic; companies with same name discarded based on location		
some companies might not actually match none of the 5 suggestions		
issue still with words appearing written together vs separated by spaces, dots, dashes etc		
tried some VBA to help with matches by email, website against company names but did not work		
checked outputs and seems that there are still matches to be resolved as they are not correctly identified		
		
		
used this that I created and tweaked with chatgpt to get the matches for all distinct rows; then dealt with the null matches separately		
WITH preprocessed AS (		
  SELECT		
    a.input_row_key,		
    a.input_company_name,		
    a.input_main_country,		
    b.company_name,		
    b.main_country,		
    LOWER(a.input_company_name) AS input_clean,		
    LOWER(b.company_name) AS company_clean,		
    REGEXP_REPLACE(LOWER(a.input_company_name), '[^a	z0	9 ]', '') AS input_alpha,
    REGEXP_REPLACE(LOWER(b.company_name), '[^a	z0	9 ]', '') AS company_alpha
  FROM company_name a		
  LEFT JOIN company_name b		
    ON a.input_row_key = b.input_row_key		
    AND (		
      b.main_country = a.input_main_country		
      OR b.main_country IS NULL		
    )		
),		
scored_matches AS (		
  SELECT		
    *,		
    		 Number match
    CASE		
      WHEN REGEXP_REPLACE(input_clean, '[^0	9]', '') = REGEXP_REPLACE(company_clean, '[^0	9]', '')
           AND REGEXP_REPLACE(input_clean, '[^0	9]', '') != ''	
      THEN 1 ELSE 0		
    END AS number_match,		
		
    		 Token score
    (		
      SELECT COUNT(*)		
      FROM (		
        SELECT iw.word		
        FROM JSON_TABLE(		
          CONCAT('["', REPLACE(input_alpha, ' ', '","'), '"]'),		
          '$[*]' COLUMNS (word VARCHAR(100) PATH '$')		
        ) AS iw		
        WHERE iw.word IN (		
          SELECT cw.word		
          FROM JSON_TABLE(		
            CONCAT('["', REPLACE(company_alpha, ' ', '","'), '"]'),		
            '$[*]' COLUMNS (word VARCHAR(100) PATH '$')		
          ) AS cw		
        )		
      ) AS common_words		
    ) AS token_score		
  FROM preprocessed		
),		
ranked_matches AS (		
  SELECT *,		
    ROW_NUMBER() OVER (		
      PARTITION BY input_row_key		
      ORDER BY number_match DESC, token_score DESC		
    ) AS rn		
  FROM scored_matches		
),		
input_keys AS (		
  SELECT DISTINCT input_row_key, input_company_name, input_main_country		
  FROM company_name		
),		
final_result AS (		
  SELECT		
    i.input_row_key,		
    i.input_company_name,		
    i.input_main_country,		
    m.company_name,		
    m.main_country,		
    m.number_match,		
    m.token_score		
  FROM input_keys i		
  LEFT JOIN ranked_matches m		
    ON i.input_row_key = m.input_row_key AND m.rn = 1		
)		
		
SELECT *		
FROM final_result		
ORDER BY input_row_key;		
		
		
		
		
used this to select only the null matches and treat them using a separate score than the split by words (treat by how they sound)		
		
WITH input_keys AS (		
  SELECT 7 AS input_row_key UNION ALL		
  SELECT 8 UNION ALL		
  SELECT 26 UNION ALL		
  SELECT 32 UNION ALL		
  SELECT 35 UNION ALL		
  SELECT 40 UNION ALL		
  SELECT 60 UNION ALL		
  SELECT 75 UNION ALL		
  SELECT 81 UNION ALL		
  SELECT 120 UNION ALL		
  SELECT 162 UNION ALL		
  SELECT 176 UNION ALL		
  SELECT 208 UNION ALL		
  SELECT 219 UNION ALL		
  SELECT 222 UNION ALL		
  SELECT 249 UNION ALL		
  SELECT 255 UNION ALL		
  SELECT 269 UNION ALL		
  SELECT 273 UNION ALL		
  SELECT 283 UNION ALL		
  SELECT 285 UNION ALL		
  SELECT 297 UNION ALL		
  SELECT 307 UNION ALL		
  SELECT 339 UNION ALL		
  SELECT 354 UNION ALL		
  SELECT 355 UNION ALL		
  SELECT 368 UNION ALL		
  SELECT 429 UNION ALL		
  SELECT 445 UNION ALL		
  SELECT 459 UNION ALL		
  SELECT 468 UNION ALL		
  SELECT 470 UNION ALL		
  SELECT 476 UNION ALL		
  SELECT 482 UNION ALL		
  SELECT 483 UNION ALL		
  SELECT 514 UNION ALL		
  SELECT 515 UNION ALL		
  SELECT 532 UNION ALL		
  SELECT 549 UNION ALL		
  SELECT 558 UNION ALL		
  SELECT 559 UNION ALL		
  SELECT 560 UNION ALL		
  SELECT 586		
),		
		
cleaned_data AS (		
  SELECT		
    k.input_row_key,		
    c.input_company_name,		
    c.input_main_country,		
    c.company_name,		
    c.main_country,		
    c.website_url,		
    c.website_domain,		
    c.primary_email,		
    c.emails,		
		
    SOUNDEX(input_company_name) AS input_sx,		
    SOUNDEX(company_name) AS company_sx		
  FROM input_keys k		
  JOIN forsql c ON k.input_row_key = c.input_row_key		
)		
		
SELECT *		
FROM (		
  SELECT *,		
    ROW_NUMBER() OVER (		
      PARTITION BY input_row_key		
      ORDER BY 		
        CASE 		
          WHEN input_sx = company_sx THEN 1		
          ELSE 2		
        END,		
        LENGTH(company_name) 		 Prefer shorter matches if tied
    ) AS rn		
  FROM cleaned_data		
  WHERE input_sx = company_sx		
     OR company_name LIKE CONCAT('%', LEFT(input_company_name, 4), '%')		
     OR website_domain LIKE CONCAT('%', LEFT(input_company_name, 4), '%')		
) ranked		
WHERE rn = 1		
ORDER BY input_row_key;		
