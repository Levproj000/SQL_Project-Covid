--SELECT Data on the table
SELECT *
FROM dbo.CovidDeaths
WHERE continent is not null
ORDER BY 1,2 


--Looking for total cases vs total Deaths
SELECT location,
	   date,
	   total_cases,
	   new_cases,
	   total_deaths,
	   population,
	   (Total_deaths/population)*100  AS DeathPercentage
FROM dbo.CovidDeaths
ORDER BY 1,2

--SHOW what population got covide

SELECT location,
	   date,
	   total_cases,
	   new_cases,
	   total_deaths,
	   population,
	   ROUND((CAST(total_cases AS float) / NULLIF(CAST(population AS float), 0)) * 100, 4) AS PercentPopulationInfected
FROM dbo.CovidDeaths
WHERE --Location like '%States%'
continent NOT IN ('World','European Union','International')
ORDER BY 1,2

-- Looking for countries with Hight infection rate compared to population

SELECT location,
	   population,
	   MAX(Total_cases) AS highest_infection_count,
	   MAX(total_cases/population)*100 AS Percent_population_infected
FROM dbo.CovidDeaths
GROUP BY location,population
ORDER BY Percent_population_infected DESC


--Showing countries with Highest Death Count Per Countries
SELECT location,
	   MAX(CAST(Total_deaths AS int)) AS Total_death_count
FROM dbo.CovidDeaths
WHERE continent is not null
GROUP BY Location
ORDER BY Total_death_count DESC;



--Breaking down by continent
SELECT continent,
	   MAX(CAST(Total_deaths AS int)) AS Total_death_count
FROM dbo.CovidDeaths
WHERE continent is not null
GROUP BY continent
ORDER BY Total_death_count DESC;


--showing continets by  with highest death count per population

SELECT continent,
	   MAX(CAST(Total_deaths AS int)) AS Total_death_count
FROM dbo.CovidDeaths
WHERE continent is not null
GROUP BY continent
ORDER BY Total_death_count DESC;


--GLOBAL NUMBER
SELECT  SUM(new_cases) as total_cases,
		SUM(CAST(new_deaths as int)) AS total_deaths,
		SUM(new_deaths)/SUM(new_cases)*100 AS Death_percentage
FROM dbo.CovidDeaths   
WHERE continent is not null
-- Group by date
ORDER BY 1,2


--JOINING TABLE

SELECT dea.continent,
	   dea.location,
	   CAST(dea.date AS date),
	   dea.population,
	   vac.new_vaccinations,
	   SUM(vac.new_vaccinations) OVER (
						PARTITION BY dea.location 
						ORDER BY dea.location) 
						AS Rolling_people_vaccinated
FROM dbo.CovidDeaths AS Dea
JOIN dbo.CovidVaccinations AS vac
ON dea.location = vac.location
and dea.date = vac.date
WHERE dea.continent is not null
ORDER BY 2,3


;WITH Popvsvac (continent, location, date, population, New_vaccinations, Rolling_people_vaccinated)
AS
(
SELECT dea.continent,
	   dea.location,
	   CAST(dea.date AS date),
	   dea.population,
	   vac.new_vaccinations,
	   SUM(CAST(vac.new_vaccinations AS bigint)) OVER ( -- Cast to bigint to prevent numeric overflow
						PARTITION BY dea.location 
						ORDER BY dea.date) -- Fix: Order by date for true rolling total
						AS Rolling_people_vaccinated
FROM dbo.CovidDeaths AS Dea
JOIN dbo.CovidVaccinations AS vac
ON dea.location = vac.location
and dea.date = vac.date
WHERE dea.continent is not null
)

-- Fix: Multiply by 100.0 first to avoid MS SQL integer division (getting 0)
SELECT *, (Rolling_people_vaccinated * 100.0 / population) AS Popvac
FROM Popvsvac;


-- INSERT INTO TEMP TABLE
DROP TABLE IF EXISTS #PercentPopvaccinated;
CREATE TABLE  #PercentPopvaccinated
(
Continent nvarchar(255),
Location varchar(255),
Date date,
population numeric,
New_vaccinations numeric,
Rolling_people_vaccinated bigint
);

INSERT INTO #PercentPopvaccinated
SELECT dea.continent,
	   dea.location,
	   CAST(dea.date AS date),
	   dea.population,
	   vac.new_vaccinations,
	   SUM(CAST(vac.new_vaccinations AS bigint)) OVER ( -- Cast to bigint to prevent numeric overflow
						PARTITION BY dea.location 
						ORDER BY dea.date) -- Order by date for true rolling total
						AS Rolling_people_vaccinated
FROM dbo.CovidDeaths AS Dea
JOIN dbo.CovidVaccinations AS vac
ON dea.location = vac.location
and dea.date = vac.date
WHERE dea.continent is not null
--ORDER BY 2,3


SELECT *, (Rolling_people_vaccinated/population)*100 AS rollvacc
FROM #PercentPopvaccinated



--CREATE A VIEW for  PERCENT POPULATION 
DROP VIEW IF EXISTS Percentpopvaccinated;
GO
CREATE VIEW Percentpopvaccinated AS
SELECT dea.continent,
	   dea.location,
	   CAST(dea.date AS date) AS date,
	   dea.population,
	   vac.new_vaccinations,
	   SUM(CAST(vac.new_vaccinations AS bigint)) OVER ( -- Cast to bigint to prevent numeric overflow
						PARTITION BY dea.location 
						ORDER BY dea.date) -- Order by date for true rolling total
						AS Rolling_people_vaccinated
FROM dbo.CovidDeaths AS Dea
JOIN dbo.CovidVaccinations AS vac
ON dea.location = vac.location
and dea.date = vac.date
WHERE dea.continent is not null
--ORDER BY 2,3
GO

SELECT *,(Rolling_people_vaccinated/population) * 100 AS popvac
FROM Percentpopvaccinated
