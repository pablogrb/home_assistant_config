# Updated setup

Started a new server project in Jan 2025. Runs on a RPi5 booting from an NVME drive on a DeskPi Lite Pi5

## Hardware setup

The case, Pi and NVME drive were installed according to DeskPi's instructions.

After flashing Raspberry Pi OS on an SD card, the Pi was booted and set up to boot from the NVME drive following Jeff Geerling's instructions [here](https://www.jeffgeerling.com/blog/2023/nvme-ssd-boot-raspberry-pi-5)

## Docker setup

Docker was installed following the official Docker docs instructions to [install using the convenience script](https://docs.docker.com/engine/install/raspberry-pi-os/#install-using-the-convenience-script). Then follow the instructions to [manage docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user) and [configure Docker to start on boot with systemd](https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot-with-systemd).

## Containers

### Portainer

Spin up a portainer container using docker compose following Winnie Ondara's instructions [here](https://www.cherryservers.com/blog/portainer-docker-compose)
