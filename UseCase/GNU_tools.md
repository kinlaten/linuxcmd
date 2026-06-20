# Grep

```sh
#Find by content: -r: recursive
grep -r "<content>" <target-dir>

```

# Find
```sh
# Bulk substitute: with sed
## +: make run once rather than sequential by each file
## -i: save change directly to original file, remove this to get console output only
find . -type f -name "<expression>" -exec sed -i 's/<target-text>/<replacement>/g' {} +

```