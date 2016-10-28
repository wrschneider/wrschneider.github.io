---
layout: post
title: Defining new metrics in Microstrategy with R integration
---

Microstrategy has a new [R Integration Pack](https://rintegrationpack.codeplex.com/) which allows you to use R scripts to calculate new Microstrategy metrics.  Here are some tips on how to use it.  I also have a [sample R script that will call the `qicharts` package](https://gist.github.com/wrschneider/f7613a5dcddac91c209315a9c6f9117c). 

*Metric formula in MSTR*

You invoke R from MSTR by defining a metric using the `RScript` function.  The barest minimum formula to invoke R is something like

```
RScript<_RScriptFile="Script.r">(Input, Input....)
```

where `_RScriptFile` is the name of the R script you want to run, and Inputs are names of MSTR metrics to pass to the R script.

*R Script inputs and outputs*

Microstrategy invokes the R script and passes metrics by defining variables,  A header in the R script tells the MSTR integration pack what variables to define in R before running the script:

```
#MICROSTRATEGY_BEGIN
#
#DESC This script takes 2 metrics from MSTR, makes a dataframe and adds them.
#DESC Revision A
#
#RVAR ParamName -parameter NumericParam1
#RVAR Metric1 -input -numeric -vector
#RVAR Metric2 -input -numeric -vector
#
#RVAR MetricSum -output -numeric -vector
#MICROSTRATEGY_END
```

This header says that two metrics can be passed to the script as inputs, and will be exposed as vectors in R, as variables named Metric1 and Metric2.  These vectors can be turned into dataframes or used 
directly.  Vectors will be ordered as they appear in the report, or can be sorted explicitly with the SortyBy parameter: e.g., `SortBy=(Month)`.  If you specify `-repeat` for the final input variable, the variable in R will be a matrix instead of a vector, with all subsequent inputs as matrix components.

The header also specifies what R variable is returned to MSTR as a metric.  Your script can have multiple potential outputs, and the output can be selected via the `_OutputVar` parameter to RScript within the angle brackets: `RScript<..., _OutputVar="RVariableName">(...)`.

Parameters are also defined in the header, and passed within the angle brackets: `RScript<_Params="ParamName=xxx">`.  For example if you are using the qicharts package for control charts, you could pass the chart type (P, U, etc.) as a parameter, and expose the control limits (CL, UCL, LCL) as individual MSTR metrics by defining them as individual output vectors and selecting one with `_OutputVar`.

*Boilerplate for Package Dependencies, Error Handling and Testing*

The R scripts provided by MSTR on the script "shelf" all have the same boilerplate for error handling, R package dependencies, and testing outside MSTR.

```
mstr.ErrMsg <- tryCatch({                                      #tryCatch for Exception Handling
  if(exists("mstr.WorkingDir")) setwd(mstr.WorkingDir)         #Working Directory if executed by MicroStrategy
 
#Check to see if package(s) are installed, install if not and then load
  CheckInstallPackages <- function(pkgs) {                     #pkgs is a vector of strings with length >= 1
    x <- lapply(pkgs, function(pkg){                           #For each pkg in pkgs (attempt to load each package one at a time):
      if(!do.call("require", list(pkg))) {                     #  Load the package if available,
        try(install.packages(pkg, lib=.Library,
                             repos="http://cran.rstudio.com")) #    Silently attempt to install into the default library
        tryCatch(do.call("library", list(pkg)),                #    Now attempt to load the package, catch error if it wasn't installed
          error = function(err) {                              #    Catch if we're unable to install into the default library
            if(!interactive()) {                               #      If non-interactive, install into this user's personal library
              personalLibPath <- Sys.getenv("R_LIBS_USER")     #        Get the path to this user's personal library
              if(is.na(match(personalLibPath, .libPaths()))) { #        If the personal library is not in the list of libraries
                dir.create(personalLibPath, recursive = TRUE)  #          Then create the personal library
                .libPaths(personalLibPath)                     #          And add the personal library to the list of libraries
              }
              install.packages(pkg, lib=personalLibPath,       #        Attempt to install the package into the personal library
                              repos="http://cran.rstudio.com") #          if this fails, raise the error back to the report
              do.call("library", list(pkg))                    #        Finally, attempt to load the package
            }
          }
        )
      }
    })
  }
  #Get the data
  if(exists("mstr.ExFlag")) {                                  #If this is executed by MicroStrategy
    # you can use inputs passed in from MSTR
  } else {                                                     #If this is NOT via a MicroStrategy Report Execution
    #NOT in MSTR, define test inputs
  }
  CheckInstallPackages(")
 # YOUR CALCULATION HERE
  
  #Finish
  try(print("Success!"))                                       #Print completion message when run from the console
 
  mstr.ErrMsg <- ""                                            #If we made it here, no errors were caught
}, error = function(err) {                                     #Catch block to report an error
  try(print(err))                                              #  Print error message to console (using try to continue on any print error)
  return(err$message)                                          #  Return error Message
})
```

*Limitations in Enterprise mode*

R visualizations are not useful in an enterprise setting.  R integration with MSTR is only useful if you can think of the output of your R script as another metric in MSTR, equivalent to an additional column on a grid.  This also means if you calculate multiple output vectors in the R script, as is common with control charts or forecasting, you will end up calling the R script multiple times from each MSTR metric.


