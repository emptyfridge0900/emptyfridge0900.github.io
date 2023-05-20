+++
title="Kanban"
date=2018-04-20

[taxonomies]
categories = ["Assignment"]
tags = ["project", "C#","SQL",".NET","WinForm"]
+++

# Project

**Advanced SQL Project Assignment**

This is a SQL-based simulation that use `triggers`, `stored procedures`, `stored functions` and `views` to model business/production line workflow.
This project consists of 5 small programs, which are

- Configuration Tool
- Workstation Andon
- Workstation Simulation
- Assembly Line Kanban
- Working Simulation (Runner)

### 1. Configuration Tool
Configuration tool can config order amount,running interval of runner, and number of stations
![configuration](/images/screenshot/configuration.png)

### 2. Workstation Simulation
Where to assemble parts
##### Assemble a completed fog lamp
- Experienced worker : 60 seconds, +/- 10%.
- New employees : 50% longer than experienced worker.
- Very experienced (super) workers : 15% less than experienced worker.
##### Average defect rates for lamps based on the workers’ skill levels
- New/Rookie 0.85%
- Experienced/Normal 0.5%
- Experienced/Super 0.15%

![simulation](/images/screenshot/simulation.png)

### 3. Workstation
Graphic representation of the part counts and status of a single workstationn
![andon](/images/screenshot/andon.png)

### 4. Assembly Line Kanban
To display the status of the entire assembly line. This displays:
- Order amount
- Passed amount
- In process amount
- Number produced 
- Yield

![kanban](/images/screenshot/kanban.png)

### 5. Working Simulation (Runner)
Simulation of a runner. A runner picks up all cards from the tray every 5 minutes. Runner goes to the stock room, picks up a new bin of the specific part and location on the card, and then goes to the station and replaces it with the old one.

![runner](/images/screenshot/runner.png)


## Getting Started

### Prerequisites

- .Net core
- Sql Server  
- Visual studio


### How to use
- Execute the queries in **Schema.sql** file to create tables
- Execute the queries in **Functions.sql** file to create stored procedures and functions
- Execute **Assembly Line** to let the workers to assemble the parts
- Execute the **Runner**. If the runner program is not running, the empty bin cannot be replaced

- Execute the **Workstation simulation** to check how many parts are left in the bins of each station
- Execute the **Workstation Andon** to monitor production status


# Tech/framework used
- [.NET](https://docs.microsoft.com/en-us/dotnet/)
- [Sql Server](https://www.microsoft.com/en-us/sql-server/sql-server-2019)
- [WinForm](https://docs.microsoft.com/en-us/dotnet/desktop/winforms/?view=netframeworkdesktop-4.8)