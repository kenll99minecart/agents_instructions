# Structure
Your code should be highly modularized. Each python file if not the main script should not have lines exceeding 2500 lines. Every folder should have an output folder, a plot folder with all the plotting functions and multiple folders for modules. Look at the plotting folder for details. Always have a .env file for important keys/api keys and yamls files for constants.

## Yaml file
**File format.** Do not make each yaml file too long. i.e. each yaml file ideally should cover a workspace/workflow. The file should not have long comments. Each comment should have a maximum of 2 lines. The yaml variables and structure should be self-explainable

**Common variables in yaml** If there is plotting function, the yaml files should standardize plotting colour type, such as cell type colours in the yaml file. Cell type scheme should also be in the yaml file as well.