---
layout: post
title: Bridge the Gap - Bring your Software to life with Elixir and Nerves
description: Writing code is fun, but nothing tops seeing your code moving things in the real world. That’s why I tested out the Nerves library.
date:   2019-07-31 13:01:35 +0200
tags:   [development, nerves, elixir]
---

Writing code is fun, but nothing tops seeing your code moving things in the real world. That’s why I tested out the [Nerves](https://nerves-project.org/) library and used $20 of Raspberry Pi utensils to let software wave a flag for me.

{% include youtube-player.html id="jterXt63tYM" %}

## The Hardware
For this project, I used the following hardware:
* Raspberry Pi Zero W
* 28BYJ-48 5v Stepper Motor with ULN2003 Board
* 16GB microSD Card
* A flag
* Some cables

## The Wiring
Here’s an overview of how to connect the ULN2003 to the Raspberry Pi.

![Wiring of ULN2003 Board to Raspberry Pi Zero W]({{site.baseurl}}/images/posts/rotate_a_flag_with_nerves/pic_1.png)

After you connect the ULN2003 Board to the Pi, connect the Stepper Motor to the ULN2003 and the Pi to your Computer. **IMPORTANT**: Connect the Pi to your Computer using the USB port and **not** the PWR port. Otherwise, you will not be able to transfer or execute code from your Computer on the Pi.

Also, make sure to connect the wires properly. The LEDs on the ULN2003 Board should glow red once you start the stepper motor.

## Setting up the Project
You can find the full code on GitHub. Here’s how you set up the project from scratch.
If you haven’t already, first install the Nerves dependencies for your operating system

```
# On macOS, run:
brew update
brew install fwup squashfs coreutils xz

# On Linux, run:
sudo apt install build-essential automake \
     autoconf git squashfs-tools ssh-askpass
```

Then install the Nerves Bootstrap library

```bash
mix archive.install hex nerves_bootstrap
```

Next, create a new Nerves project and install its dependencies 
with

```bash
mix nerves.new move_it
cd move_it

```

Add the circuits_gpio dependency to your mix.exs file

```elixir

defp deps do
  [
    ...
    {:circuits_gpio, "~> 0.4.1"},
    ...
  ]
end
```

Install the dependencies with

```bash
export MIX_TARGET=rpi0
mix deps.get
```

You should now have a fully set up project. As a last step, enable the logger by changing the `rootfs_overlay/etc/iex.exs` like this:

```elixir
if RingLogger in Application.get_env(:logger, :backends, []) do
  IO.puts("""
  ...
  """)
  RingLogger.attach() # <- Add this line
end
```

## The Code

Open the `lib/move_it/move_it.ex` file and fill it with the following code:

```elixir
defmodule MoveIt do
  @moduledoc """
  Documentation for MoveIt.
  """

  require Logger

  alias Circuits.GPIO

  @pins [5, 6, 13, 26]

  def start(count) do
    pins =
      Enum.reduce(@pins, [], fn pin, acc ->
        Logger.info("Starting pin #{pin} as output")
        {:ok, gpio} = GPIO.open(pin, :output)
        acc ++ gpio
      end)

    Logger.info("Starting the motor. Hold on to your butts!")

    spawn(fn -> step(count, pins) end)
    {:ok, self()}
  end

  defp step(0, pins) do
    Logger.info("End reached. Closing the pin connections...")

    for pin <- pins do
      pin
      |> GPIO.write(0)
      |> GPIO.close()
    end

    Logger.info("Pin connections closed. Good bye.")
  end

  defp step(round, pins) do
    Logger.info("Round Nr: #{round}")

    next_idx = rem(round, length(pins))
    prev_idx = if next_idx == 0, do: length(pins) - 1, else: next_idx - 1

    next_pin = Enum.at(pins, next_idx)
    prev_pin = Enum.at(pins, prev_idx)

    GPIO.write(prev_pin, 0)
    GPIO.write(next_pin, 1)

    Process.sleep(2)
    step(round - 1, pins)
  end
end
```

The code above first opens the GPIO pins with the GPIO.open/2 function and then starts the motor by sequentially enabling and disabling the Pins. This causes the stepper motor to move one step forward each time one of the pins is enabled. You can adjust how many steps the motor will make by defining the `count` parameter when starting the script with `MoveIt.start(count)`

## Setting up the Raspberry Pi

First, insert the SD card into your computer.

Now, bundle the code into a firmware and copy it to the SD Card with the commands

```bash
mix firmware
mix firmware.burn
```

Nerves “should” be able to find your SD card and will ask you to confirm it. If it doesn’t find your SD card, you can supply the mount point with the `-d` option like so:

```bash
# For example
mix firmware.burn -d /dev/rdisk2
```

After the copy process is completed, insert the SD Card into your Raspberry Pi and connect the Pi with the USB cable to your computer.

## Getting Things moving

Now, the thrilling part comes. Connect your computer to the Raspberry Pi with

```bash
ssh nerves.local
```

This will start an `iex` terminal on the Raspberry Pi. Now, start the motor with

```elixir
MoveIt.start(1200)
```

The motor should turn, you should be filled with awe, and in the distance you will hear the appreciative clapping of Richard Stallman for using open-source software :)