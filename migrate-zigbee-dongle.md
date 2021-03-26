
# How to change zigbee dongle with zigbee2mqtt

Disclaimer: **Re-pairing all devices again is required**.

Steps:
 
  1. Stop zigbee2mqtt daemon/container
  2. Unplug old USB dongle and plug the new one (make sure is correctly detected)
  3. Go to the zigbee2mqtt folder `/opt/zigbee2mqtt/`
  4. Remove device database (a new one will be created) with `rm data/database.db`
  5. Edit configuration with `vi data/configuration.yaml`
  6. Adapt/Change the serial port device if needed. In my case the new one is `/dev/ttyUSB0`
  7. Change the pan_id under "advanced" section adding: `pan_id: "0x1a63"`
  8. Set the `permit_join` to true
  9. Save with `:wq` 😉
  10. Start zigbee2mqtt again

Remember to change `permit_join` to false after repairing.
