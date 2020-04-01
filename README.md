# EV-Sales
SQL HDDBD Database build for SAP HANA


 
/*  CreateCDS HDTable- file:  fga.hdbdd  */
/* Associations and keys are not necessary for import and for tables to join */
/* On import, primary key (X1) set- State for .Elec, Local for .Stat, STATE for Demo */
/* on hdbti import, tables created with populated data for .Elec and .Stat */
/* .Demo showed duplicate row error, so key dropped from STATE-table imports */

namespace GBI_807;

@Schema: 'GBI_807'

context fga {
			
    	  	entity  Elec {
    		    key 	   State: String(30);
            			EVSales2016: Integer;
            			EVSales2017: Integer;
			EVSales2018: Integer;
    		};
	
    		entity  Stat {

    key        Locale: String(30);
    	Sid: Association [0..1] to Elec on Sid.State=Locale;
            			Population2015: Integer;
		    key	   Abbreviation: String(2);
    		};
    		entity Demo	 {
		    	Did: Association [1..*] to Stat on Did.Abbreviation=STATE;
		    	STATE: String(2);
            			Zip: String(5);
            			NumberReturns: Integer;
			Dependents: Integer;
            			Elderly: Integer;
            			AGI: Integer;
            			TotalIncome: Integer;
			NoRETaxes: Integer;
			EstateTaxAmount: Integer;
			NoBusinessIncomes: Integer;
    		};
}

 
import = [
            {
                        table = "GBI_807::fga.Elec";
                        schema = "GBI_807";
                        file = "GBI_807:importelec.csv";
                        header = true;
                        delimField = ",";
            }
];
 
import = [
            {
                        table = "GBI_807::fga.Stat";
                        schema = "GBI_807";
                        file = "GBI_807:importstat.csv";
                        header = true;
                        delimField = ",";
            }
];
 
import = [
            {
                        table = "GBI_807::fga.Demo";
                        schema = "GBI_807";
                        file = "GBI_807:d.csv";
                        header = true;
                        delimField = ",";
            }
];


 

CREATE COLUMN TABLE "GBI_807"."GBI_807::group.EV" AS (SELECT * FROM  "GBI_807"."GBI_807::fga.Elec");  /* creates new tables */
CREATE COLUMN TABLE "GBI_807"."GBI_807::group.SP" AS (SELECT * FROM  "GBI_807"."GBI_807::fga.Stat");  /* good to have backup */
CREATE COLUMN TABLE "GBI_807"."GBI_807::group.DD" AS (SELECT * FROM  "GBI_807"."GBI_807::fga.Demo"); /* in case mess up */


/* To delete aggregated numbers for states in Demographic Data-we can calculate */
DELETE FROM "GBI_807"."GBI_807::group.DD"				
WHERE "Zip" IN ('00000', '99999');
/* 1. List the top 10 states by total electric vehicle sales (descending order) */

 
/* 2016 */
SELECT TOP 10 "State", "EVSales2016" AS EV2016
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2016"
ORDER BY EV2016 DESC;
 
/* 2017 */
SELECT TOP 10 "State", "EVSales2017" AS EV2017
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2017"
ORDER BY EV2017 DESC;
 

/* 2018 */
SELECT TOP 10 "State", "EVSales2018" AS EV2018
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2018"
ORDER BY EV2018 DESC;

 
/* 2016-2018 Combined */
SELECT TOP 10 "State", ("EVSales2016"+"EVSales2017"+"EVSales2018") AS TotalEV
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018"
ORDER BY TotalEV DESC;












/* 2. List the top 10 states by number of electric vehicle sales per capita (descending order) */
 
/* 2016 */
SELECT TOP 10 "State", ("EVSales2016"/"Population2015") AS PERCAPITA16
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "Population2015"
ORDER BY PERCAPITA16 DESC;
 
/* 2017 */
SELECT TOP 10 "State", ("EVSales2017"/"Population2015") AS PERCAPITA17
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2017", "Population2015"
ORDER BY PERCAPITA17 DESC;
 
/* 2018 */
SELECT TOP 10 "State", ("EVSales2018"/"Population2015") AS PERCAPITA18
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2018", "Population2015"
ORDER BY PERCAPITA18 DESC;
 
/* To get Per Capita per year averaged */
SELECT TOP 10 "State", (("EVSales2016"+"EVSales2017"+"EVSales2018")/("Population2015"*3)) AS PERCAPITAAVG
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018", "Population2015"
ORDER BY PERCAPITAAVG DESC;
 
/* To get percap over 3Y */
SELECT TOP 10 "State", (("EVSales2016"+"EVSales2017"+"EVSales2018")/"Population2015") AS PERCAPITA3Y
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018", "Population2015"
ORDER BY PERCAPITA3Y DESC;
















/* 3. List for states beginning with 'A', the zip codes with total incomes greater than the average for the state */
 
SELECT DD2."STATE", DD2."Zip", CONCAT ('$ ',TO_DECIMAL(DD2."TotalIncome", 10, 2)) AS TOTINC, 
CONCAT ('$ ',TO_DECIMAL(DD1."AVGINC", 10, 2)) AS AVGINC /* To remove AVGINC from results, remove DD1."AVGINC here */
FROM "GBI_807"."GBI_807::group.DD" AS DD2 
, (SELECT "STATE", SUM("TotalIncome")/COUNT(DISTINCT "Zip") AS AVGINC, AVG("TotalIncome") AS AVGTOTAL 
FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
)
AS DD1
WHERE DD2."STATE" LIKE ('A%') AND DD2."STATE" = DD1."STATE"
AND DD2."TotalIncome">DD1."AVGINC";

/* SEE ADDENDUM FOR ALL 412 Rows */







/* 4. List for each state, the top zip code by AGI?  */
 
/* Including Washington DC */
SELECT DD1."STATE", DD1."Zip", CONCAT ('$ ',TO_DECIMAL(DD2."TOPAGI", 10, 2)) AS TOPAGI
FROM "GBI_807"."GBI_807::group.DD" AS DD1
INNER JOIN (SELECT "STATE",MAX(AGI) AS TOPAGI
    FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
) AS DD2
ON DD1."STATE" = DD2."STATE" 
AND DD1."AGI" = DD2."TOPAGI";

 


/* Partition By method with DC */
with cte as (
 select
  row_number() over (partition by "STATE" order by "AGI" desc) as rownumber,
  "STATE",
  "Zip",
  CONCAT ('$ ',TO_DECIMAL("AGI", 10, 2)) AS TOPAGI
 from "GBI_807"."GBI_807::group.DD"
)
select * from cte where rownumber = 1;

 
/* Not Including Washington DC */
SELECT DD1."STATE", DD1."Zip",CONCAT ('$ ',TO_DECIMAL(DD2."TOPAGI", 10, 2)) AS TOPAGI
FROM "GBI_807"."GBI_807::group.DD" AS DD1
INNER JOIN (SELECT "STATE",MAX(AGI) AS TOPAGI
    FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
) AS DD2
ON DD1."STATE" = DD2."STATE" 
AND DD1."AGI" = DD2."TOPAGI"
WHERE NOT DD1."STATE" IN ('DC');
 
/* partition by method no DC  */
with cte as (
 select
  row_number() over (partition by "STATE" order by "AGI" desc) as rownumber,
  "STATE",
  "Zip",
  CONCAT ('$ ',TO_DECIMAL("AGI", 10, 2)) AS TOPAGI
 from "GBI_807"."GBI_807::group.DD"
 WHERE NOT "STATE" IN ('DC')
)
select * from cte where rownumber = 1;

/* SEE ADDENDUM FOR FULL 50+51 ZIP Code RESULTS */

/* 5. List the top 5 states by number of returns per capita (descending order) */
 
/* Including Washington DC */
SELECT TOP 5 DD."STATE", SUM(DD."NumberReturns"/ SP."Population2015") AS RTNCAP
FROM "GBI_807"."GBI_807::group.SP" AS SP INNER JOIN "GBI_807"."GBI_807::group.DD" AS DD ON SP."Abbreviation"= DD."STATE"
GROUP BY DD."STATE"
ORDER BY RTNCAP DESC;

 
/* Not Including Washington DC */
SELECT TOP 5 DD."STATE", SUM(DD."NumberReturns"/ SP."Population2015") AS RTNCAP
FROM "GBI_807"."GBI_807::group.SP" AS SP INNER JOIN "GBI_807"."GBI_807::group.DD" AS DD ON SP."Abbreviation"= DD."STATE"
WHERE NOT DD."STATE" IN ('DC')
GROUP BY DD."STATE"
ORDER BY RTNCAP DESC;

1. Which states have the most electric vehicle sales (total and over time)?

 






2. How would electric vehicle sales totals look on a map of states?
 

 




3. Which states have the most electric vehicle sales per capita (total and over time)?
 


















4. Show car sales over 3 years for 10 states that have the highest median AGI
With DC

 






Without DC
 


5. Are there any significant relationships between state electric vehicle sales, state populations and average state incomes?

This part shows taking the AVG of Total Income for each state versus the SUM of Total Income for each state divided by the Number of returns for each state (as discuss in email)

 
This shows relationships between Total EV Sales vs. Average State Income vs State Populations

 
 

 

 
 

California is the Outlier in Yellow
 



APPENDIX




QUESTION 3 SQL Full 412 Row Table Results
	STATE	Zip	TOTINC	AVGINC
1	AL	35004	$283,221.00 	$194,519.83 
2	AL	35007	$701,514.00 	$194,519.83 
3	AL	35010	$382,646.00 	$194,519.83 
4	AL	35016	$361,470.00 	$194,519.83 
5	AL	35020	$259,289.00 	$194,519.83 
6	AL	35022	$552,470.00 	$194,519.83 
7	AL	35023	$484,788.00 	$194,519.83 
8	AL	35040	$399,598.00 	$194,519.83 
9	AL	35043	$390,635.00 	$194,519.83 
10	AL	35045	$256,336.00 	$194,519.83 
11	AL	35051	$204,773.00 	$194,519.83 
12	AL	35055	$413,210.00 	$194,519.83 
13	AL	35057	$273,337.00 	$194,519.83 
14	AL	35058	$216,214.00 	$194,519.83 
15	AL	35071	$456,043.00 	$194,519.83 
16	AL	35077	$220,744.00 	$194,519.83 
17	AL	35080	$548,303.00 	$194,519.83 
18	AL	35094	$344,443.00 	$194,519.83 
19	AL	35111	$441,344.00 	$194,519.83 
20	AL	35115	$233,852.00 	$194,519.83 
21	AL	35120	$314,475.00 	$194,519.83 
22	AL	35121	$300,507.00 	$194,519.83 
23	AL	35124	$915,969.00 	$194,519.83 
24	AL	35126	$513,336.00 	$194,519.83 
25	AL	35127	$225,090.00 	$194,519.83 
26	AL	35128	$243,334.00 	$194,519.83 
27	AL	35146	$277,018.00 	$194,519.83 
28	AL	35150	$301,304.00 	$194,519.83 
29	AL	35160	$387,795.00 	$194,519.83 
30	AL	35173	$1,038,614.00 	$194,519.83 
31	AL	35180	$318,723.00 	$194,519.83 
32	AL	35205	$428,236.00 	$194,519.83 
33	AL	35206	$201,104.00 	$194,519.83 
34	AL	35209	$1,134,655.00 	$194,519.83 
35	AL	35210	$396,253.00 	$194,519.83 
36	AL	35211	$341,170.00 	$194,519.83 
37	AL	35213	$1,502,478.00 	$194,519.83 
38	AL	35214	$314,661.00 	$194,519.83 
39	AL	35215	$700,467.00 	$194,519.83 
40	AL	35216	$1,479,816.00 	$194,519.83 
41	AL	35222	$314,460.00 	$194,519.83 
42	AL	35223	$1,903,031.00 	$194,519.83 
43	AL	35226	$1,619,591.00 	$194,519.83 
44	AL	35235	$400,175.00 	$194,519.83 
45	AL	35242	$3,359,972.00 	$194,519.83 
46	AL	35243	$1,512,470.00 	$194,519.83 
47	AL	35244	$1,546,699.00 	$194,519.83 
48	AL	35401	$331,038.00 	$194,519.83 
49	AL	35404	$333,424.00 	$194,519.83 
50	AL	35405	$875,050.00 	$194,519.83 
51	AL	35406	$858,315.00 	$194,519.83 
52	AL	35453	$236,801.00 	$194,519.83 
53	AL	35473	$401,708.00 	$194,519.83 
54	AL	35475	$513,756.00 	$194,519.83 
55	AL	35504	$313,527.00 	$194,519.83 
56	AL	35565	$203,707.00 	$194,519.83 
57	AL	35601	$630,043.00 	$194,519.83 
58	AL	35603	$878,339.00 	$194,519.83 
59	AL	35611	$474,922.00 	$194,519.83 
60	AL	35613	$678,179.00 	$194,519.83 
61	AL	35630	$557,166.00 	$194,519.83 
62	AL	35633	$441,672.00 	$194,519.83 
63	AL	35634	$348,858.00 	$194,519.83 
64	AL	35640	$575,726.00 	$194,519.83 
65	AL	35645	$368,641.00 	$194,519.83 
66	AL	35650	$250,496.00 	$194,519.83 
67	AL	35653	$196,531.00 	$194,519.83 
68	AL	35661	$466,837.00 	$194,519.83 
69	AL	35674	$395,301.00 	$194,519.83 
70	AL	35748	$239,211.00 	$194,519.83 
71	AL	35749	$741,906.00 	$194,519.83 
72	AL	35750	$295,415.00 	$194,519.83 
73	AL	35756	$559,520.00 	$194,519.83 
74	AL	35757	$533,533.00 	$194,519.83 
75	AL	35758	$1,854,632.00 	$194,519.83 
76	AL	35759	$256,482.00 	$194,519.83 
77	AL	35761	$307,410.00 	$194,519.83 
78	AL	35763	$799,052.00 	$194,519.83 
79	AL	35768	$228,044.00 	$194,519.83 
80	AL	35769	$276,406.00 	$194,519.83 
81	AL	35773	$277,516.00 	$194,519.83 
82	AL	35801	$1,184,174.00 	$194,519.83 
83	AL	35802	$984,398.00 	$194,519.83 
84	AL	35803	$978,788.00 	$194,519.83 
85	AL	35805	$225,437.00 	$194,519.83 
86	AL	35806	$774,520.00 	$194,519.83 
87	AL	35810	$449,348.00 	$194,519.83 
88	AL	35811	$745,655.00 	$194,519.83 
89	AL	35824	$309,713.00 	$194,519.83 
90	AL	35901	$417,728.00 	$194,519.83 
91	AL	35903	$276,596.00 	$194,519.83 
92	AL	35904	$197,007.00 	$194,519.83 
93	AL	35906	$236,741.00 	$194,519.83 
94	AL	35907	$234,116.00 	$194,519.83 
95	AL	35950	$345,183.00 	$194,519.83 
96	AL	35957	$259,544.00 	$194,519.83 
97	AL	35967	$262,865.00 	$194,519.83 
98	AL	35976	$432,165.00 	$194,519.83 
99	AL	36022	$349,667.00 	$194,519.83 
100	AL	36027	$256,462.00 	$194,519.83 
101	AL	36037	$232,381.00 	$194,519.83 
102	AL	36054	$333,267.00 	$194,519.83 
103	AL	36064	$436,157.00 	$194,519.83 
104	AL	36066	$523,472.00 	$194,519.83 
105	AL	36067	$578,753.00 	$194,519.83 
106	AL	36078	$270,889.00 	$194,519.83 
107	AL	36081	$276,096.00 	$194,519.83 
108	AL	36092	$422,507.00 	$194,519.83 
109	AL	36093	$408,767.00 	$194,519.83 
110	AL	36106	$499,677.00 	$194,519.83 
111	AL	36108	$221,565.00 	$194,519.83 
112	AL	36109	$567,459.00 	$194,519.83 
113	AL	36111	$398,490.00 	$194,519.83 
114	AL	36116	$757,627.00 	$194,519.83 
115	AL	36117	$1,614,930.00 	$194,519.83 
116	AL	36203	$419,608.00 	$194,519.83 
117	AL	36207	$501,891.00 	$194,519.83 
118	AL	36265	$362,888.00 	$194,519.83 
119	AL	36301	$663,628.00 	$194,519.83 
120	AL	36303	$732,394.00 	$194,519.83 
121	AL	36305	$571,890.00 	$194,519.83 
122	AL	36330	$852,250.00 	$194,519.83 
123	AL	36360	$348,456.00 	$194,519.83 
124	AL	36420	$225,707.00 	$194,519.83 
125	AL	36426	$256,086.00 	$194,519.83 
126	AL	36502	$254,207.00 	$194,519.83 
127	AL	36507	$381,199.00 	$194,519.83 
128	AL	36526	$1,050,774.00 	$194,519.83 
129	AL	36527	$564,284.00 	$194,519.83 
130	AL	36530	$200,404.00 	$194,519.83 
131	AL	36532	$1,097,184.00 	$194,519.83 
132	AL	36535	$515,821.00 	$194,519.83 
133	AL	36541	$290,989.00 	$194,519.83 
134	AL	36542	$355,890.00 	$194,519.83 
135	AL	36551	$201,253.00 	$194,519.83 
136	AL	36561	$294,102.00 	$194,519.83 
137	AL	36567	$253,917.00 	$194,519.83 
138	AL	36571	$386,731.00 	$194,519.83 
139	AL	36575	$428,166.00 	$194,519.83 
140	AL	36582	$474,690.00 	$194,519.83 
141	AL	36587	$202,475.00 	$194,519.83 
142	AL	36604	$212,018.00 	$194,519.83 
143	AL	36605	$383,288.00 	$194,519.83 
144	AL	36606	$347,575.00 	$194,519.83 
145	AL	36608	$1,189,331.00 	$194,519.83 
146	AL	36609	$499,982.00 	$194,519.83 
147	AL	36618	$302,312.00 	$194,519.83 
148	AL	36619	$354,247.00 	$194,519.83 
149	AL	36693	$530,227.00 	$194,519.83 
150	AL	36695	$1,424,343.00 	$194,519.83 
151	AL	36701	$367,300.00 	$194,519.83 
152	AL	36801	$479,467.00 	$194,519.83 
153	AL	36804	$359,039.00 	$194,519.83 
154	AL	36830	$1,254,812.00 	$194,519.83 
155	AL	36832	$346,645.00 	$194,519.83 
156	AL	36853	$195,253.00 	$194,519.83 
157	AL	36854	$268,022.00 	$194,519.83 
158	AL	36867	$379,361.00 	$194,519.83 
159	AL	36869	$278,859.00 	$194,519.83 
160	AL	36870	$335,612.00 	$194,519.83 
161	AL	36877	$246,388.00 	$194,519.83 
162	AR	71601	$160,993.00 	$135,790.69 
163	AR	71602	$337,766.00 	$135,790.69 
164	AR	71603	$575,528.00 	$135,790.69 
165	AR	71635	$220,217.00 	$135,790.69 
166	AR	71655	$265,364.00 	$135,790.69 
167	AR	71671	$144,129.00 	$135,790.69 
168	AR	71701	$326,190.00 	$135,790.69 
169	AR	71730	$800,310.00 	$135,790.69 
170	AR	71753	$293,915.00 	$135,790.69 
171	AR	71801	$219,904.00 	$135,790.69 
172	AR	71822	$151,623.00 	$135,790.69 
173	AR	71832	$154,226.00 	$135,790.69 
174	AR	71852	$169,413.00 	$135,790.69 
175	AR	71854	$717,738.00 	$135,790.69 
176	AR	71901	$593,027.00 	$135,790.69 
177	AR	71909	$530,039.00 	$135,790.69 
178	AR	71913	$1,034,535.00 	$135,790.69 
179	AR	71923	$290,464.00 	$135,790.69 
180	AR	71953	$233,320.00 	$135,790.69 
181	AR	72002	$401,218.00 	$135,790.69 
182	AR	72007	$194,082.00 	$135,790.69 
183	AR	72012	$278,627.00 	$135,790.69 
184	AR	72015	$556,988.00 	$135,790.69 
185	AR	72019	$755,023.00 	$135,790.69 
186	AR	72022	$438,548.00 	$135,790.69 
187	AR	72023	$902,738.00 	$135,790.69 
188	AR	72032	$637,320.00 	$135,790.69 
189	AR	72034	$1,216,425.00 	$135,790.69 
190	AR	72058	$394,081.00 	$135,790.69 
191	AR	72076	$653,162.00 	$135,790.69 
192	AR	72086	$213,776.00 	$135,790.69 
193	AR	72103	$240,148.00 	$135,790.69 
194	AR	72104	$369,475.00 	$135,790.69 
195	AR	72110	$234,898.00 	$135,790.69 
196	AR	72112	$151,340.00 	$135,790.69 
197	AR	72113	$899,478.00 	$135,790.69 
198	AR	72116	$742,401.00 	$135,790.69 
199	AR	72117	$194,277.00 	$135,790.69 
200	AR	72118	$392,295.00 	$135,790.69 
201	AR	72120	$945,449.00 	$135,790.69 
202	AR	72143	$719,777.00 	$135,790.69 
203	AR	72150	$279,206.00 	$135,790.69 
204	AR	72160	$253,084.00 	$135,790.69 
205	AR	72173	$214,182.00 	$135,790.69 
206	AR	72176	$167,276.00 	$135,790.69 
207	AR	72202	$274,108.00 	$135,790.69 
208	AR	72204	$424,688.00 	$135,790.69 
209	AR	72205	$682,503.00 	$135,790.69 
210	AR	72206	$364,088.00 	$135,790.69 
211	AR	72207	$924,013.00 	$135,790.69 
212	AR	72209	$364,589.00 	$135,790.69 
213	AR	72210	$404,702.00 	$135,790.69 
214	AR	72211	$830,077.00 	$135,790.69 
215	AR	72212	$806,084.00 	$135,790.69 
216	AR	72223	$1,633,065.00 	$135,790.69 
217	AR	72227	$512,846.00 	$135,790.69 
218	AR	72301	$356,380.00 	$135,790.69 
219	AR	72315	$350,462.00 	$135,790.69 
220	AR	72335	$175,047.00 	$135,790.69 
221	AR	72364	$389,425.00 	$135,790.69 
222	AR	72396	$254,142.00 	$135,790.69 
223	AR	72401	$1,039,724.00 	$135,790.69 
224	AR	72404	$824,426.00 	$135,790.69 
225	AR	72450	$709,543.00 	$135,790.69 
226	AR	72455	$208,530.00 	$135,790.69 
227	AR	72501	$470,509.00 	$135,790.69 
228	AR	72543	$267,035.00 	$135,790.69 
229	AR	72560	$136,971.00 	$135,790.69 
230	AR	72601	$524,624.00 	$135,790.69 
231	AR	72616	$176,503.00 	$135,790.69 
232	AR	72653	$589,240.00 	$135,790.69 
233	AR	72701	$983,373.00 	$135,790.69 
234	AR	72703	$1,085,754.00 	$135,790.69 
235	AR	72704	$773,636.00 	$135,790.69 
236	AR	72712	$4,948,326.00 	$135,790.69 
237	AR	72714	$404,210.00 	$135,790.69 
238	AR	72715	$509,403.00 	$135,790.69 
239	AR	72718	$139,384.00 	$135,790.69 
240	AR	72719	$330,981.00 	$135,790.69 
241	AR	72730	$228,966.00 	$135,790.69 
242	AR	72734	$141,626.00 	$135,790.69 
243	AR	72736	$140,422.00 	$135,790.69 
244	AR	72740	$163,588.00 	$135,790.69 
245	AR	72745	$303,484.00 	$135,790.69 
246	AR	72751	$150,123.00 	$135,790.69 
247	AR	72753	$188,371.00 	$135,790.69 
248	AR	72756	$942,343.00 	$135,790.69 
249	AR	72758	$1,601,440.00 	$135,790.69 
250	AR	72761	$444,801.00 	$135,790.69 
251	AR	72762	$1,090,600.00 	$135,790.69 
252	AR	72764	$997,426.00 	$135,790.69 
253	AR	72801	$253,587.00 	$135,790.69 
254	AR	72802	$548,611.00 	$135,790.69 
255	AR	72830	$244,584.00 	$135,790.69 
256	AR	72834	$170,452.00 	$135,790.69 
257	AR	72837	$167,197.00 	$135,790.69 
258	AR	72901	$324,165.00 	$135,790.69 
259	AR	72903	$715,547.00 	$135,790.69 
260	AR	72904	$237,997.00 	$135,790.69 
261	AR	72908	$359,883.00 	$135,790.69 
262	AR	72916	$350,933.00 	$135,790.69 
263	AR	72921	$270,414.00 	$135,790.69 
264	AR	72936	$366,658.00 	$135,790.69 
265	AR	72949	$167,048.00 	$135,790.69 
266	AR	72956	$610,723.00 	$135,790.69 
267	AZ	85008	$801,661.00 	$587,192.91 
268	AZ	85013	$664,538.00 	$587,192.91 
269	AZ	85014	$757,118.00 	$587,192.91 
270	AZ	85016	$1,675,150.00 	$587,192.91 
271	AZ	85018	$2,374,087.00 	$587,192.91 
272	AZ	85020	$1,153,614.00 	$587,192.91 
273	AZ	85021	$1,012,065.00 	$587,192.91 
274	AZ	85022	$1,384,648.00 	$587,192.91 
275	AZ	85023	$888,118.00 	$587,192.91 
276	AZ	85024	$791,923.00 	$587,192.91 
277	AZ	85027	$863,987.00 	$587,192.91 
278	AZ	85028	$1,028,811.00 	$587,192.91 
279	AZ	85029	$779,911.00 	$587,192.91 
280	AZ	85032	$1,608,158.00 	$587,192.91 
281	AZ	85033	$611,143.00 	$587,192.91 
282	AZ	85037	$715,888.00 	$587,192.91 
283	AZ	85041	$906,191.00 	$587,192.91 
284	AZ	85042	$827,476.00 	$587,192.91 
285	AZ	85044	$1,599,615.00 	$587,192.91 
286	AZ	85048	$1,715,934.00 	$587,192.91 
287	AZ	85050	$1,391,893.00 	$587,192.91 
288	AZ	85051	$617,914.00 	$587,192.91 
289	AZ	85083	$804,055.00 	$587,192.91 
290	AZ	85085	$930,133.00 	$587,192.91 
291	AZ	85086	$1,464,508.00 	$587,192.91 
292	AZ	85122	$881,505.00 	$587,192.91 
293	AZ	85138	$767,878.00 	$587,192.91 
294	AZ	85140	$731,695.00 	$587,192.91 
295	AZ	85142	$1,695,634.00 	$587,192.91 
296	AZ	85143	$665,221.00 	$587,192.91 
297	AZ	85201	$739,064.00 	$587,192.91 
298	AZ	85202	$822,604.00 	$587,192.91 
299	AZ	85203	$796,974.00 	$587,192.91 
300	AZ	85204	$993,013.00 	$587,192.91 
301	AZ	85205	$1,037,062.00 	$587,192.91 
302	AZ	85206	$761,466.00 	$587,192.91 
303	AZ	85207	$1,634,986.00 	$587,192.91 
304	AZ	85208	$606,996.00 	$587,192.91 
305	AZ	85209	$975,204.00 	$587,192.91 
306	AZ	85210	$626,868.00 	$587,192.91 
307	AZ	85212	$886,109.00 	$587,192.91 
308	AZ	85213	$1,125,944.00 	$587,192.91 
309	AZ	85224	$1,386,617.00 	$587,192.91 
310	AZ	85225	$1,644,789.00 	$587,192.91 
311	AZ	85226	$1,482,481.00 	$587,192.91 
312	AZ	85233	$1,242,299.00 	$587,192.91 
313	AZ	85234	$1,815,012.00 	$587,192.91 
314	AZ	85248	$1,650,814.00 	$587,192.91 
315	AZ	85249	$1,981,034.00 	$587,192.91 
316	AZ	85250	$807,331.00 	$587,192.91 
317	AZ	85251	$1,810,698.00 	$587,192.91 
318	AZ	85253	$3,560,948.00 	$587,192.91 
319	AZ	85254	$2,459,698.00 	$587,192.91 
320	AZ	85255	$4,531,297.00 	$587,192.91 
321	AZ	85257	$785,575.00 	$587,192.91 
322	AZ	85258	$1,618,341.00 	$587,192.91 
323	AZ	85259	$1,889,642.00 	$587,192.91 
324	AZ	85260	$2,217,921.00 	$587,192.91 
325	AZ	85262	$1,515,629.00 	$587,192.91 
326	AZ	85266	$975,054.00 	$587,192.91 
327	AZ	85268	$1,196,095.00 	$587,192.91 
328	AZ	85281	$981,467.00 	$587,192.91 
329	AZ	85282	$1,246,811.00 	$587,192.91 
330	AZ	85283	$1,194,111.00 	$587,192.91 
331	AZ	85284	$1,261,453.00 	$587,192.91 
332	AZ	85286	$1,701,255.00 	$587,192.91 
333	AZ	85295	$1,476,885.00 	$587,192.91 
334	AZ	85296	$1,468,146.00 	$587,192.91 
335	AZ	85297	$1,142,786.00 	$587,192.91 
336	AZ	85298	$1,426,868.00 	$587,192.91 
337	AZ	85301	$655,169.00 	$587,192.91 
338	AZ	85302	$691,209.00 	$587,192.91 
339	AZ	85304	$661,882.00 	$587,192.91 
340	AZ	85308	$2,049,651.00 	$587,192.91 
341	AZ	85310	$789,521.00 	$587,192.91 
342	AZ	85323	$627,620.00 	$587,192.91 
343	AZ	85326	$1,001,457.00 	$587,192.91 
344	AZ	85331	$1,455,589.00 	$587,192.91 
345	AZ	85338	$1,270,401.00 	$587,192.91 
346	AZ	85339	$905,957.00 	$587,192.91 
347	AZ	85340	$1,026,591.00 	$587,192.91 
348	AZ	85345	$1,060,614.00 	$587,192.91 
349	AZ	85351	$595,851.00 	$587,192.91 
350	AZ	85353	$623,251.00 	$587,192.91 
351	AZ	85364	$1,287,414.00 	$587,192.91 
352	AZ	85365	$959,879.00 	$587,192.91 
353	AZ	85374	$1,009,556.00 	$587,192.91 
354	AZ	85375	$840,894.00 	$587,192.91 
355	AZ	85379	$1,021,894.00 	$587,192.91 
356	AZ	85381	$763,026.00 	$587,192.91 
357	AZ	85382	$1,296,690.00 	$587,192.91 
358	AZ	85383	$2,148,172.00 	$587,192.91 
359	AZ	85388	$627,849.00 	$587,192.91 
360	AZ	85392	$860,956.00 	$587,192.91 
361	AZ	85395	$1,055,071.00 	$587,192.91 
362	AZ	85614	$661,981.00 	$587,192.91 
363	AZ	85629	$676,569.00 	$587,192.91 
364	AZ	85635	$765,986.00 	$587,192.91 
365	AZ	85641	$805,670.00 	$587,192.91 
366	AZ	85704	$1,035,837.00 	$587,192.91 
367	AZ	85705	$603,071.00 	$587,192.91 
368	AZ	85710	$1,093,883.00 	$587,192.91 
369	AZ	85711	$722,139.00 	$587,192.91 
370	AZ	85712	$626,944.00 	$587,192.91 
371	AZ	85715	$655,074.00 	$587,192.91 
372	AZ	85716	$670,221.00 	$587,192.91 
373	AZ	85718	$2,011,831.00 	$587,192.91 
374	AZ	85719	$627,581.00 	$587,192.91 
375	AZ	85730	$740,142.00 	$587,192.91 
376	AZ	85737	$977,010.00 	$587,192.91 
377	AZ	85739	$628,489.00 	$587,192.91 
378	AZ	85741	$762,252.00 	$587,192.91 
379	AZ	85742	$797,406.00 	$587,192.91 
380	AZ	85743	$851,808.00 	$587,192.91 
381	AZ	85745	$939,484.00 	$587,192.91 
382	AZ	85746	$623,407.00 	$587,192.91 
383	AZ	85747	$732,397.00 	$587,192.91 
384	AZ	85748	$676,572.00 	$587,192.91 
385	AZ	85749	$967,117.00 	$587,192.91 
386	AZ	85750	$1,473,384.00 	$587,192.91 
387	AZ	85755	$790,376.00 	$587,192.91 
388	AZ	86001	$784,390.00 	$587,192.91 
389	AZ	86004	$969,727.00 	$587,192.91 
390	AZ	86301	$641,829.00 	$587,192.91 
391	AZ	86305	$725,866.00 	$587,192.91 
392	AZ	86314	$640,960.00 	$587,192.91 
393	AK	99501	$640,556.00 	$389,999.00 
394	AK	99502	$1,086,686.00 	$389,999.00 
395	AK	99503	$517,361.00 	$389,999.00 
396	AK	99504	$1,186,170.00 	$389,999.00 
397	AK	99507	$1,587,097.00 	$389,999.00 
398	AK	99508	$942,822.00 	$389,999.00 
399	AK	99515	$1,068,994.00 	$389,999.00 
400	AK	99516	$1,488,457.00 	$389,999.00 
401	AK	99517	$682,670.00 	$389,999.00 
402	AK	99577	$1,194,184.00 	$389,999.00 
403	AK	99611	$488,077.00 	$389,999.00 
404	AK	99615	$417,500.00 	$389,999.00 
405	AK	99645	$994,409.00 	$389,999.00 
406	AK	99654	$1,034,440.00 	$389,999.00 
407	AK	99669	$612,816.00 	$389,999.00 
408	AK	99701	$493,105.00 	$389,999.00 
409	AK	99705	$618,593.00 	$389,999.00 
410	AK	99709	$841,897.00 	$389,999.00 
411	AK	99801	$882,488.00 	$389,999.00 
412	AK	99901	$419,627.00 	$389,999.00 


QUESTION 4 SQL PARTITION BY Method 50 States
	ROWNUMBER	STATE	Zip	TOPAGI
1	1	CA	94301	$10,296,266.00 
2	1	KY	41017	$2,404,558.00 
3	1	MO	63017	$2,833,698.00 
4	1	AK	99507	$1,564,350.00 
5	1	NJ	07030	$4,657,261.00 
6	1	AL	35242	$3,300,440.00 
7	1	ID	83646	$1,618,072.00 
8	1	IL	60611	$7,952,291.00 
9	1	NC	28277	$3,909,028.00 
10	1	PA	19087	$3,483,611.00 
11	1	WV	26508	$1,363,247.00 
12	1	FL	33480	$6,887,326.00 
13	1	KS	66062	$2,717,450.00 
14	1	MA	02116	$5,563,788.00 
15	1	MI	48103	$2,756,404.00 
16	1	ND	58104	$1,630,855.00 
17	1	SC	29464	$2,680,516.00 
18	1	TN	37027	$4,658,577.00 
19	1	VT	05403	$826,051.00 
20	1	HI	96816	$2,070,480.00 
21	1	DE	19711	$1,785,082.00 
22	1	IN	46032	$3,392,527.00 
23	1	NE	68516	$1,881,267.00 
24	1	NM	87111	$2,248,500.00 
25	1	OH	45040	$2,707,826.00 
26	1	SD	57108	$1,590,154.00 
27	1	CT	06830	$7,242,398.00 
28	1	RI	02906	$1,509,876.00 
29	1	GA	30327	$4,953,938.00 
30	1	MS	39110	$2,175,211.00 
31	1	NV	89052	$3,417,061.00 
32	1	TX	77024	$8,684,531.00 
33	1	WA	98004	$4,469,557.00 
34	1	WI	53217	$2,909,176.00 
35	1	WY	82009	$1,343,206.00 
36	1	LA	70810	$2,092,957.00 
37	1	MD	20854	$6,370,883.00 
38	1	MT	59102	$1,478,630.00 
39	1	OR	97229	$3,952,039.00 
40	1	UT	84020	$2,015,237.00 
41	1	CO	80134	$3,227,230.00 
42	1	IA	52722	$1,778,811.00 
43	1	ME	04401	$1,075,069.00 
44	1	OK	73013	$2,761,652.00 
45	1	NH	03110	$1,531,391.00 
46	1	AZ	85255	$4,451,010.00 
47	1	AR	72712	$4,928,494.00 
48	1	MN	55391	$2,723,441.00 
49	1	NY	10021	$13,597,373.00 
50	1	VA	22101	$5,032,226.00 

QUESTION 4 SQL PARTITION BY Method 51 States (including Washington DC)


	ROWNUMBER	STATE	Zip	TOPAGI
1	1	CA	94301	$10,296,266.00 
2	1	KY	41017	$2,404,558.00 
3	1	MO	63017	$2,833,698.00 
4	1	AK	99507	$1,564,350.00 
5	1	NJ	07030	$4,657,261.00 
6	1	AL	35242	$3,300,440.00 
7	1	ID	83646	$1,618,072.00 
8	1	IL	60611	$7,952,291.00 
9	1	NC	28277	$3,909,028.00 
10	1	PA	19087	$3,483,611.00 
11	1	WV	26508	$1,363,247.00 
12	1	FL	33480	$6,887,326.00 
13	1	KS	66062	$2,717,450.00 
14	1	MA	02116	$5,563,788.00 
15	1	MI	48103	$2,756,404.00 
16	1	ND	58104	$1,630,855.00 
17	1	SC	29464	$2,680,516.00 
18	1	TN	37027	$4,658,577.00 
19	1	VT	05403	$826,051.00 
20	1	HI	96816	$2,070,480.00 
21	1	DE	19711	$1,785,082.00 
22	1	IN	46032	$3,392,527.00 
23	1	NE	68516	$1,881,267.00 
24	1	NM	87111	$2,248,500.00 
25	1	OH	45040	$2,707,826.00 
26	1	SD	57108	$1,590,154.00 
27	1	CT	06830	$7,242,398.00 
28	1	RI	02906	$1,509,876.00 
29	1	GA	30327	$4,953,938.00 
30	1	MS	39110	$2,175,211.00 
31	1	NV	89052	$3,417,061.00 
32	1	TX	77024	$8,684,531.00 
33	1	WA	98004	$4,469,557.00 
34	1	WI	53217	$2,909,176.00 
35	1	WY	82009	$1,343,206.00 
36	1	LA	70810	$2,092,957.00 
37	1	MD	20854	$6,370,883.00 
38	1	MT	59102	$1,478,630.00 
39	1	OR	97229	$3,952,039.00 
40	1	UT	84020	$2,015,237.00 
41	1	CO	80134	$3,227,230.00 
42	1	IA	52722	$1,778,811.00 
43	1	ME	04401	$1,075,069.00 
44	1	OK	73013	$2,761,652.00 
45	1	NH	03110	$1,531,391.00 
46	1	AZ	85255	$4,451,010.00 
47	1	AR	72712	$4,928,494.00 
48	1	MN	55391	$2,723,441.00 
49	1	NY	10021	$13,597,373.00 
50	1	VA	22101	$5,032,226.00 
51	1	DC	20016	$3,882,614.00 
				


FULL SQL CODE

CREATE COLUMN TABLE "GBI_807"."GBI_807::group.EV" AS (SELECT * FROM  "GBI_807"."GBI_807::g6.EV");  /* creates new tables */
CREATE COLUMN TABLE "GBI_807"."GBI_807::group.SP" AS (SELECT * FROM  "GBI_807"."GBI_807::g6.SP");  /* good to have backup */
CREATE COLUMN TABLE "GBI_807"."GBI_807::group.DD" AS (SELECT * FROM  "GBI_807"."GBI_807::g6.DD"); /* in case mess up */


/* To delete aggregated numbers for states in Demographic Data-we can calculate */
DELETE FROM "GBI_807"."GBI_807::group.DD"				
WHERE "Zip" IN ('00000', '99999');

/* 1. List the top 10 states by total electric vehicle sales (descending order) */

/* 2016 */
SELECT TOP 10 "State", "EVSales2016" AS EV2016
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2016"
ORDER BY EV2016 DESC;

/* 2017 */
SELECT TOP 10 "State", "EVSales2017" AS EV2017
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2017"
ORDER BY EV2017 DESC;

/* 2018 */
SELECT TOP 10 "State", "EVSales2018" AS EV2018
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2018"
ORDER BY EV2018 DESC;

/* 2016-2018 Combined */
SELECT TOP 10 "State", ("EVSales2016"+"EVSales2017"+"EVSales2018") AS TotalEV
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018"
ORDER BY TotalEV DESC;

/* 2. List the top 10 states by number of electric vehicle sales per capita (descending order) */

/* 2016 */
SELECT TOP 10 "State", ("EVSales2016"/"Population2015") AS PERCAPITA16
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "Population2015"
ORDER BY PERCAPITA16 DESC;

/* 2017 */
SELECT TOP 10 "State", ("EVSales2017"/"Population2015") AS PERCAPITA17
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2017", "Population2015"
ORDER BY PERCAPITA17 DESC;

/* 2018 */
SELECT TOP 10 "State", ("EVSales2018"/"Population2015") AS PERCAPITA18
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2018", "Population2015"
ORDER BY PERCAPITA18 DESC;

/* To get Per Capita per year averaged */
SELECT TOP 10 "State", (("EVSales2016"+"EVSales2017"+"EVSales2018")/("Population2015"*3)) AS PERCAPITAAVG
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018", "Population2015"
ORDER BY PERCAPITAAVG DESC;

/* To get percap over 3Y */
SELECT TOP 10 "State", (("EVSales2016"+"EVSales2017"+"EVSales2018")/"Population2015") AS PERCAPITA3Y
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"=SP."Locale"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018", "Population2015"
ORDER BY PERCAPITA3Y DESC;

/* 3. List for states beginning with 'A', the zip codes with total incomes greater than the average for the state */

SELECT DD2."STATE", DD2."Zip", CONCAT ('$ ',TO_DECIMAL(DD2."TotalIncome", 10, 2)) AS TOTINC, 
CONCAT ('$ ',TO_DECIMAL(DD1."AVGINC", 10, 2)) AS AVGINC /* To remove AVGINC from results, remove DD1."AVGINC here */
FROM "GBI_807"."GBI_807::group.DD" AS DD2 
, (SELECT "STATE", SUM("TotalIncome")/COUNT(DISTINCT "Zip") AS AVGINC, AVG("TotalIncome") AS AVGTOTAL 
FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
)
AS DD1
WHERE DD2."STATE" LIKE ('A%') AND DD2."STATE" = DD1."STATE"
AND DD2."TotalIncome">DD1."AVGINC";

/* 4. List for each state, the top zip code by AGI?  */



/* Including Washington DC */
SELECT DD1."STATE", DD1."Zip", CONCAT ('$ ',TO_DECIMAL(DD2."TOPAGI", 10, 2)) AS TOPAGI
FROM "GBI_807"."GBI_807::group.DD" AS DD1
INNER JOIN (SELECT "STATE",MAX(AGI) AS TOPAGI
    FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
) AS DD2
ON DD1."STATE" = DD2."STATE" 
AND DD1."AGI" = DD2."TOPAGI";

/* Partition By method with DC */

with cte as (
 select
  row_number() over (partition by "STATE" order by "AGI" desc) as rownumber,
  "STATE",
  "Zip",
  CONCAT ('$ ',TO_DECIMAL("AGI", 10, 2)) AS TOPAGI
 from "GBI_807"."GBI_807::group.DD"
)
select * from cte where rownumber = 1;

/* Not Including Washington DC */
SELECT DD1."STATE", DD1."Zip",CONCAT ('$ ',TO_DECIMAL(DD2."TOPAGI", 10, 2)) AS TOPAGI
FROM "GBI_807"."GBI_807::group.DD" AS DD1
INNER JOIN (SELECT "STATE",MAX(AGI) AS TOPAGI
    FROM "GBI_807"."GBI_807::group.DD"
    GROUP BY "STATE"
) AS DD2
ON DD1."STATE" = DD2."STATE" 
AND DD1."AGI" = DD2."TOPAGI"
WHERE NOT DD1."STATE" IN ('DC');

/* partition by method no DC  */
with cte as (
 select
  row_number() over (partition by "STATE" order by "AGI" desc) as rownumber,
  "STATE",
  "Zip",
  CONCAT ('$ ',TO_DECIMAL("AGI", 10, 2)) AS TOPAGI
 from "GBI_807"."GBI_807::group.DD"
 WHERE NOT "STATE" IN ('DC')
)
select * from cte where rownumber = 1;

/* 5. List the top 5 states by number of returns per capita (descending order) */

/* Including Washington DC */
SELECT TOP 5 DD."STATE", SUM(DD."NumberReturns"/ SP."Population2015") AS RTNCAP
FROM "GBI_807"."GBI_807::group.SP" AS SP INNER JOIN "GBI_807"."GBI_807::group.DD" AS DD ON SP."Abbreviation"= DD."STATE"
GROUP BY DD."STATE"
ORDER BY RTNCAP DESC;

/* Not Including Washington DC */
SELECT TOP 5 DD."STATE", SUM(DD."NumberReturns"/ SP."Population2015") AS RTNCAP
FROM "GBI_807"."GBI_807::group.SP" AS SP INNER JOIN "GBI_807"."GBI_807::group.DD" AS DD ON SP."Abbreviation"= DD."STATE"
WHERE NOT DD."STATE" IN ('DC')
GROUP BY DD."STATE"
ORDER BY RTNCAP DESC;

/*  Trying *******************************************   This is Tableau that I have by doing in SQL.  We do'nt necessarily need to submit.  1+3 should be correct
4 I am working on and may be able to get now with partition by and now that I know median exists */

/* 1. Which states have the most electric vehicle sales (total and over time)? */
/* with DC */
SELECT "State", "EVSales2016", "EVSales2017", "EVSales2018", SUM("EVSales2016"+"EVSales2017"+"EVSales2018") AS TOTAL
FROM "GBI_807"."GBI_807::group.EV"
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018"
ORDER BY TOTAL DESC;

/* without DC */
SELECT "State", "EVSales2016", "EVSales2017", "EVSales2018", SUM("EVSales2016"+"EVSales2017"+"EVSales2018") AS TOTAL
FROM "GBI_807"."GBI_807::group.EV"
WHERE NOT "State" IN ('District of Columbia')
GROUP BY "State", "EVSales2016", "EVSales2017", "EVSales2018"
ORDER BY TOTAL DESC;

/* Tableau-3. Which states have the most electric vehicle sales per capita (total and over time)? */
SELECT EV."State", SUM(EV."EVSales2016"/SP."Population2015") AS PC16, SUM(EV."EVSales2017"/SP."Population2015") AS PC17,
SUM(EV."EVSales2018"/SP."Population2015") AS PC18, SUM(EV."EVSales2016"+EV."EVSales2017"+EV."EVSales2018")/(SP."Population2015") AS PCOT
FROM "GBI_807"."GBI_807::group.EV" AS EV INNER JOIN "GBI_807"."GBI_807::group.SP" AS SP ON EV."State"= SP."Locale"
GROUP BY EV."State", SP."Population2015"  /* PCOT is sum of Percap for '16-'18 */
ORDER BY PCOT DESC, PC16 DESC, PC17 DESC, PC18 DESC;

SELECT "STATE" AS ST, "NoBusinessIncomes" AS "#BizInc", "AGI", SUM(COUNT "Zip")
FROM "GBI_807"."GBI_807::group.DD" AS DD
GROUP BY "STATE", "AGI", "NoBusinessIncomes", "Zip"
ORDER BY "#BizInc" DESC;

/* 4. Show car sales over 3 years for 10 states that have the highest median AGI */ 
 /* Without individual, or individual business income/AGI data, it is impossible
to figure out the the median AGI within a ZIP and, therefore, state.  Any number 
from "NoBusinessIncomes" or "NumberReturns" will be irrelevant, so the median AGI from state must
be calculated by Zip Code with Median AGI in each state. */
/*
ALL BELOW trying things 
CREATE VIEW "GBI_807.MedianAGI" AS;
SELECT "STATE", (((COUNT(DISTINCT "Zip"))+1)/2) AS MEDZIP,  CONCAT('$ ',TO_DECIMAL(AVG("AGI"), 10, 2)) AS AGI
FROM "GBI_807"."GBI_807::group.DD"
GROUP BY "STATE";

SELECT "STATE", (((COUNT(DISTINCT "Zip"))+1)/2) AS MEDZIP,  CONCAT('$ ',TO_DECIMAL(AVG("AGI"), 10, 2)) AS AGI
FROM "GBI_807"."GBI_807::group.DD" AS DD1 INNER JOIN (SELECT DD1."STATE", DD1."Zip", RANK(DD1."AGI") AS AGIRANK FROM "GBI_807"."GBI_807::group.DD" GROUP BY "STATE", "Zip") AS DD2
ON DD1.MEDZIP=DD2.AGIRANK OR ((DD1.MEDZIP)+.5)=DD2.AGIRANK Or ((DD1.MEDZIP)-.5)=DD2.AGIRANK
GROUP BY "STATE", "Zip", "AGI"
ORDER BY "STATE", "AGI";

SELECT * 
FROM "GBI_807.MedianAGI";

HAVING ROWNUM COUNT(DISTINCT "Zip")/2)-1);  */


********************************************************************************************************************************************************
R for 3D Plot Visualizations Below
****************************************************************************************************************************************************




FULL R CODE setwd("/cloud/project")
data4<-read.csv("corr2.csv")
summary(data)
setrowname <- function() {
  row.names(data)=data[,1]
  data<-data[,-1]
}
setrowname(data4)
data$ev2018=round(data$ev2018, digits = -1)
summary(data)

data$percap16=data$ev2016/data$population
data$percap17=data$ev2017/data$population
data$percap18=data$ev2018/data$population
data$percaptotal=data$evtotal/data$population
data$percapavg=data$evtotal/(data$population*3)
library(scatterplot3d)
library(plot3D)
library(ggcorrplot)
library(corrplot)
library(ggplot2)
data
data(data2)
with(data2, text3D(evtotal, population, incret, 
                       labels = rownames(data2), colvar = percapavg, 
                       col = gg.col(100), theta = 60, phi = 20,
                       xlab = "Total EV Sales", ylab = "Population", zlab = "Average Income", 
                       main = "Relationships-EV Sales, State Populations, + Avg. Total Income", cex = 0.6, 
                       bty = "g", ticktype = "detailed", d = 2,
                       clab = "3Y Avg EV Sales Per Capita", adj = 0.5, font = 2))
data(data)
with(data, text3D(evtotal, incret, population))
                  labels = rownames(data), colvar = population, 
                  col = gg.col(100), theta = 60, phi = 20,
                  xlab = "Total EV Sales", ylab = "AVGINC=INC/RET", ylab = "Population",
                  main = "Relationships-EV Sales, State Populations, + Avg. Total Income", cex = 0.6, 
                  bty = "g", ticktype = "detailed", d = 2,
                  clab = "3Y Avg EV Sales Per Capita", adj = 0.5, font = 2)


# x, y, z variables
x <- data$evtotal
y <- data$population
z <- data$avginc
# Compute the linear regression (z = ax + by + d)
fit <- lm(z ~ x + y)
summary(fit)
# predict values on regular xy grid
grid.lines = 26
x.pred <- seq(min(x), max(x), length.out = grid.lines)
y.pred <- seq(min(y), max(y), length.out = grid.lines)
xy <- expand.grid( x = x.pred, y = y.pred)
z.pred <- matrix(predict(fit, newdata = xy), 
                 nrow = grid.lines, ncol = grid.lines)
# fitted points for droplines to surface
fitpoints <- predict(fit)
# scatter plot with regression plane
scatter3D(x, y, z, pch = 18, cex = 2, 
          theta = 20, phi = 20, ticktype = "detailed",
          xlab = "EV Total Sales", ylab = "Population", zlab = "Avg. Inc.",  
          surf = list(x = x.pred, y = y.pred, z = z.pred,  
                      facets = NA, fit = fitpoints), main = "Relationships-EV Sales, State Populations, + Avg. Total Income")

w=data4$state
y1=data$percap16
y2=data$percap17
y3=data$percap18
y4=data$percaptotal


# Create a first line
plot(w, y1, type = "b", frame = FALSE, pch = 19, 
     col = "red", xlab = "x", ylab = "y")
# Add a second line
lines(w, y2, pch = 18, col = "blue", type = "b", lty = 4)
lines(w, y3, pch = 18, col = "green", type = "b", lty = 4)
lines(w, y4, pch = 18, col = "black", type = "b", lty = 4)
# Add a legend to the plot
legend("topleft", legend=c("Line 1", "Line 2"),
       col=c("red", "blue"), lty = 1:2, cex=0.8)
summary(data)
cp=cor(data[, 4:6])
ggcorrplot(cp, main = "Relationships-EV Sales, State Populations, + Avg. Total Income")
cd=cor(data)
pairs(data)
data4
pairs(data4)
plot(data4$state, data4$evtotal)
hist(data$percaptotal)

row.names(data)=data[,1]
data<-data[,-1]
pairs(data5)
########   adding in sum total income and number of business incomes
setwd("/cloud/project")
data<-read.csv("corr3.csv")
summary(data)
pairs(data)
plot(x,data$nobizinc)
data<-data[,-1]
data$incret=data2$realavginc
data$returns=data2$numret
data$bizinc<-data$sumincome/data$nobizinc
plot(data$bizinc, data$avginc, main = "Sum Total Inc/# of Biz returns vs. 
     Average Total Income", xlab = "Total Income/# of Business Returns", ylab= "Avg Total Income")
pairs(data)
summary(data)
str(data)
data$percap16=data$ev2016/data$population
data$percap17=data$ev2017/data$population
data$percap18=data$ev2018/data$population
data$percaptotal=data$evtotal/data$population
data$percapavg=data$evtotal/(data$population*3)
library(scatterplot3d)
library(plot3D)
library(ggcorrplot)
library(corrplot)
library(ggplot2)

data(data)
with(data, text3D(evtotal, population, avginc, 
                  labels = rownames(data), colvar = percapavg, 
                  col = gg.col(100), theta = 60, phi = 20,
                  xlab = "Total EV Sales", ylab = "Population", zlab = "Average Income", 
                  main = "Relationships-EV Sales, State Populations, + Avg. Total Income", cex = 0.6, 
                  bty = "g", ticktype = "detailed", d = 2,
                  clab = "3Y Avg EV Sales Per Capita", adj = 0.5, font = 2))

# x, y, z variables
x <- data$evtotal
y <- data$population
z <- data$avginc
# Compute the linear regression (z = ax + by + d)
fit <- lm(z ~ x + y)
# predict values on regular xy grid
grid.lines = 26
x.pred <- seq(min(x), max(x), length.out = grid.lines)
y.pred <- seq(min(y), max(y), length.out = grid.lines)
xy <- expand.grid( x = x.pred, y = y.pred)
z.pred <- matrix(predict(fit, newdata = xy), 
                 nrow = grid.lines, ncol = grid.lines)
# fitted points for droplines to surface
fitpoints <- predict(fit)
# scatter plot with regression plane
scatter3D(x, y, z, pch = 18, cex = 2, 
          theta = 20, phi = 20, ticktype = "detailed",
          xlab = "EV Total Sales", ylab = "Population", zlab = "Avg. Inc.",  
          surf = list(x = x.pred, y = y.pred, z = z.pred,  
                      facets = NA, fit = fitpoints), main = "Relationships-EV Sales, State Populations, + Avg. Total Income")

w=data4$state
y1=data$percap16
y2=data$percap17
y3=data$percap18
y4=data$percaptotal


# Create a first line
plot(w, y1, type = "b", frame = FALSE, pch = 19, 
     col = "red", xlab = "x", ylab = "y")
# Add a second line
lines(w, y2, pch = 18, col = "blue", type = "b", lty = 4)
lines(w, y3, pch = 18, col = "green", type = "b", lty = 4)
lines(w, y4, pch = 18, col = "black", type = "b", lty = 4)
# Add a legend to the plot
legend("topleft", legend=c("Line 1", "Line 2"),
       col=c("red", "blue"), lty = 1:2, cex=0.8)
summary(data)
cp=cor(data[, 4:6])
ggcorrplot(cp, main = "Relationships-EV Sales, State Populations, + Avg. Total Income")
cd=cor(data)
pairs(data)
data4
pairs(data4)
plot(data4$state, data4$evtotal)
hist(data$percaptotal)

row.names(data4)=data4[,1]
data5<-data4[,-1]
pairs(data5)
summary(data)

data2=read.csv("retin.csv")
data2
data2$realavginc=data2$income/data2$numret
data2

