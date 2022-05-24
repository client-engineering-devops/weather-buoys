# weather-buoys

[![](https://travis-ci.org/modern-fortran/weather-buoys.svg?branch=master)](https://travis-ci.org/modern-fortran/weather-buoys)

Processing weather buoy data in parallel.

## Getting started

### Dependencies

This repo assumes you are working with GNU Fortran compiler
and an OpenCoarrays wrappers `caf` and `cafrun`.
If you are working with other coarray-enabled compiler
such as Intel or Cray compilers, edit the `FC` variable
in the Makefile.

### Getting the code

```
git clone https://github.com/modern-fortran/weather-buoys 
```

### Compiling the programs

```
cd weather-buoys
make
```

### Running the serial program

```
./weather_stats
```

If you run the program with the dataset included in this repo,
you will get the output similar to this:

```
 Maximum wind speed measured is    40.9000015     at station 42001
 Highest mean wind speed is    6.47883749     at station 42020
 Lowest mean wind speed is    5.43456125     at station 42036
```

### Running the parallel program

Run the program on 2 parallel images:

```
cafrun -n 2 ./weather_stats_parallel
```

The result should be the same as in the serial program,
but will complete somewhat faster, depending on the number 
of cores available and number of images you invoke.
You can run the program on any number of images,
but no more than the number of files in dataset (9).

## Contributors

* [Michael Hirsch](https://github.com/scivision)

# Differences for IBM Build

## Creating the docker image:

1. git clone the appropriate repo

```
git clone https://github.com/client-engineering-devops/weather-buoys.git
cd weather-buoys
docker build -t weather-buoys .
docker run -d -p 9090:9090 weather-buoys # This starts up the image we built and forwards the ports
```
If you want to get all fancy, you can move the `data` directory out of the cloned repo and tie a file mount to it in docker:

```
cp -pr data /path/somewhere/else/weather_data
chmod 777 /path/somewhere/else/weather_data

docker run -d -v /path/somwhere/else/weather_data:/data -p 9090:9090 weather-buoys
```

2. The image is also available built on quay.io
```
docker pull quay.io/kramerro_ibm/weather-buoys
docker pull quay.io/kramerro_ibm/weather-buoys:ppc64le # For the powerpc version
```

## What did we do differently?

In a nutshell, we needed to test our fortran code in a container and be able to pass arguments to it. In context, this means via REST API or HTTP paramaters. To accomplish this we first modified the two fortran files mentioned above in the original README:

```
src/weather_stats.f90
src/weather_stats_parallel.f90
```
These fortran files needed to take arguments. Once built, they could take the weather buoy id numbers as args and run their processing on the CSV files.

### Mixing in NodeJS

We created a server.js file and inserted API calls into it:

```
// This lets us pass the values in the url line as part of the get request query. Eg: URL/inputvals?buoy1=10001&buoy2=10002&buoy3=10003
// The key names are not important in our case as we are stripping them out and only passing the values along to the weather_stats fortran app

app.get('/inputvals', function(req, res){
  console.log(req.query);
  const buoys = Object.values(req.query);
  console.log(buoys);
  exec("./weather_stats_parallel " + buoys.join(" "), (error, stdout, stderr) => {
    if (error) {
      console.log(`error: ${error.message}`);
      res.end( `error: ${error.message}` );
    }
    if (stderr) {
      console.log(`stderr: ${stderr}`);
      res.end( `error: ${stderr}` );
    }
    res.end( `${stdout}` );
  })
  console.log(buoys.join(" "));
})

// This allows us to do a POST to the url and pass in a JSON to the api: {"buoys":["42002"]}

app.post('/api/buoys', function(req, res) {
  const data = (req.body.buoys);
  data.forEach(element => {
    console.log(element);
  });
  exec("./weather_stats_parallel " + data.join(" "), (error, stdout, stderr) => {
    if (error) {
      console.log(`error: ${error.message}`);
      res.end( `error: ${error.message}` );
    }
    if (stderr) {
      console.log(`stderr: ${stderr}`);
      res.end( `error: ${stderr}` );
    }
    res.end( `${stdout}` );
  })
  console.log(data.join(" "));
});

app.listen(port, () =>
	console.log(`Example app listening on port ${port}!`),
);

app.get('/weather_stats', function (req, res) {
  exec("./weather_stats_parallel", (error, stdout, stderr) => {
      if (error) {
          console.log(`error: ${error.message}`);
          res.end( `error: ${error.message}` );
      }
      if (stderr) {
          console.log(`stderr: ${stderr}`);
          res.end( `error: ${stderr}` );
      }
      console.log(`stdout: ${stdout}`);
      res.end( `${stdout}` );
  });
})
```

Now we could make a call to our URL and post a json:

```
curl -X POST http://<ip or localhost>:9090/api/buoys -H 'Content-Type: application/json' -d '{"buoys":["42002","42003","42020","42036"]}'

Maximum wind speed measured is    40.9000015     at station 42001
 Highest mean wind speed is    6.47883749     at station 42020
 Lowest mean wind speed is    5.73942709     at station 42001
```
Or we could post a parameterized call:
```
curl -X GET "http://ip or localhost>:9090/inputvals?buoy1=42002&buoy2=42003&buoy3=42020&buoy4=42036"

 Maximum wind speed measured is    40.9000015     at station 42001
 Highest mean wind speed is    6.47883749     at station 42020
 Lowest mean wind speed is    5.43456125     at station 42036
 ```

