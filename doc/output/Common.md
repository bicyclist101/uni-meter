# Common output device configuration options

The configuration options from below are common to all output devices. At the time of writing these are
the Shelly Pro3EM and the EcoTracker. The samples always show the Shelly Pro3EM configuration. For
usage with the EcoTracker please replace the ``shelly-pro3`` property with ``eco-tracker``.

## Configuring the "forget" interval

If the physical input device is not reachable and no power values are available for a certain time, the uni-meter will
not provide any output values to the storage anymore, so that the storage triggers its fallback behavior.

Without any configuration that happens after one minute. If you need a different timeout, you can configure it using 
the following configuration below:

```hocon
uni-meter {
  #...
  output-devices {
    #...
    shelly-pro3em {
      # These are the defaults used without any configuration: 
      forget-interval = 1m
    }
  }
}
```

## Configuring a static power offset

In some setups, it might be necessary to add a static offset to the power values. This can be the case if the real
electrical meter readings are not 100% accurate to your household's electrical meter readings.

You can either configure a power offset for the single phases or a total power offset. The phase power offsets take
precedence over the total power offset. If at least one phase power offset is configured, the total power offset is
ignored.

Setting the power offset is done in the `/etc/uni-meter.conf` file:

```hocon
uni-meter {
  #...
  output-devices {
    #...
    shelly-pro3em {
      #...
      power-offset-total =0
      
      power-offset-l1 = 0
      power-offset-l2 = 0
      power-offset-l3 = 0
    }
  }
}
```

## Configuring client-specific behavior

uni-meter can send individual power values to different clients. This allows the usage with multiple  
independent storages. This is more or less an experimental feature. Even if all storages only get a share
of the current power, over time small fluctuations of the charging/discharging power may lead to the
situation that one storage charges another one. That can be corrected by disabling the complete uni-meter
in certain intervals using the [REST API to switch on/off](../api/SwitchOnOff.md) for short periods, for instance, 
through a cron job. 

For each client, specified by its IP address, the MAC address and the power factor to use can be configured.
For all other clients not found in that ``client-contexts`` list a ``default-client-power-factor`` can be
set which defaults to 1.0.

```hocon
uni-meter {
  #...
  output-devices {
    shelly-pro3em {
      #...
      default-client-power-factor = 0.5
      client-contexts = [{
        # See correct data from local machine
        address = "127.0.0.1"
        mac = "abcdef0123456"
        power-factor = 1.0      
      },{
        # Small batterie
        address = "192.168.178.30"
        mac = "bcdef01234567"
        power-factor = 0.3     
      },{
        # Big batterie
        address = "192.168.178.70"
        mac = "cdef012345678"
        power-factor = 0.7     
      }]
    }
  }
}
```

## Configuring the default voltage and/or frequency

If you are using the uni-meter in a Non-European country, which has different grid voltage and frequency 
standards as Europe, you can change these parameters as below.

```hocon
uni-meter {
  #...
  output-devices {
    #...
    shelly-pro3em {
      # North american settings
      default-voltage = 115.0
      default-frequency = 60.0
    }
  }
}
```

## Configuring power averaging (exponential smoothing)

Power measurements can fluctuate rapidly, causing unwanted charging/discharging behavior in battery storage systems. 
The uni-meter provides time-based exponential smoothing to dampen these fluctuations.

You can configure separate averaging time constants for power increases and decreases on each phase. This allows 
asymmetric behavior, such as:
- **Slow response to power increases** (smoothing spikes) to prevent rapid charging bursts
- **Fast response to power decreases** (no smoothing) to quickly react to reduced available power

### How it works

The averaging uses exponential smoothing with a time constant (in seconds):
- **0s**: No smoothing, measurements are used directly
- **Larger values**: More smoothing, slower response to changes

The weight of past measurements is calculated as:
```
pastWeight = (averagingSeconds - intervalSeconds) / averagingSeconds
```

where `intervalSeconds` is the time since the last measurement. If too much time passes 
(intervalSeconds >= averagingSeconds), no smoothing is applied.

### Configuration examples

**Example 1: Smooth power increases only (prevent charging spikes)**
```hocon
uni-meter {
  #...
  output-devices {
    shelly-pro3em {
      #...
      # Smooth power increases over 10 seconds, but react immediately to decreases
      power-averaging-seconds-l1-up = 10
      power-averaging-seconds-l1-down = 0
      power-averaging-seconds-l2-up = 10
      power-averaging-seconds-l2-down = 0
      power-averaging-seconds-l3-up = 10
      power-averaging-seconds-l3-down = 0
    }
  }
}
```

**Example 2: Different smoothing per phase**
```hocon
uni-meter {
  #...
  output-devices {
    shelly-pro3em {
      #...
      # Phase 1: Heavy smoothing (solar panel with clouds)
      power-averaging-seconds-l1-up = 15
      power-averaging-seconds-l1-down = 5
      
      # Phase 2: Moderate smoothing
      power-averaging-seconds-l2-up = 5
      power-averaging-seconds-l2-down = 2
      
      # Phase 3: No smoothing (stable load)
      power-averaging-seconds-l3-up = 0
      power-averaging-seconds-l3-down = 0
    }
  }
}
```

**Example 3: Symmetric smoothing**
```hocon
uni-meter {
  #...
  output-devices {
    shelly-pro3em {
      #...
      # Smooth both increases and decreases equally
      power-averaging-seconds-l1-up = 8
      power-averaging-seconds-l1-down = 8
      power-averaging-seconds-l2-up = 8
      power-averaging-seconds-l2-down = 8
      power-averaging-seconds-l3-up = 8
      power-averaging-seconds-l3-down = 8
    }
  }
}
```

### Runtime adjustment

Like power offsets, averaging times can be adjusted at runtime via the REST API:

```bash
# Adjust smoothing times dynamically
curl "http://<uni-meter-ip>:<port>/api/set_parameters?power-averaging-seconds-l1-up=5&power-averaging-seconds-l1-down=0"
```

See [REST API to dynamically adjust runtime parameters](../api/SetParameters.md) for details.

### Recommendations

- **Solar systems with fluctuating input**: Use averaging on "up" (e.g., 10-15s) to prevent rapid charging cycles
- **Stable grid input**: Keep averaging at 0s for immediate response
- **Battery protection**: Use different up/down times to prevent aggressive charging but allow quick discharge stops
- **Start conservative**: Begin with small values (3-5s) and increase if needed
