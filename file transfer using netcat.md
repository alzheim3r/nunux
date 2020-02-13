# File transfer using netcat

When you're out of options, and if nothing else helps, maybe you can use .. nc!

### On the listening (receiving) part

```
nc -l -p 1299 > image.jpg

```

### on the sending part 

```
nc -q 0 10.0.0.10 1299 < image.jpg

```

`-q 0` is necessary or netcat won't quit after file is transfered

# Sending multiple files

### On the listening (receiving) part

```
nc -lp 1299 | tar xz

```

### on the sending part

```
tar -czf - file1 file2 | nc -q 0 10.0.0.10 1299

```


