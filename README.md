@echo off
setlocal enableextensions enabledelayedexpansion

set start=0000
set end=9999

:loop
if %start%==%end% goto endloop

for /l %%i in (!start!) do ( 
	set 
