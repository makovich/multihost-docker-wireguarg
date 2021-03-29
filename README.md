# Multihost Docker using WireGuard overlay network

Read the full blog post [here](https://makovi.ch/barges-of-serfs/).

## Requirements

* [Vagrant](https://www.vagrantup.com/)

## Usage

```sh
$ vagrant up
$ vagrant ssh barge3 -c "docker exec serf0 serf members"
```

## License

MIT/Unlicense
