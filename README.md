Siemens MRI software is structured
Author - Sameer Shinde 
<br>
<br>
import pypulseq as pp
import numpy as np

# --- System limits (Siemens-like constraints) ---
system = pp.Opts(
    max_grad=80,        # mT/m
    grad_unit='mT/m',
    max_slew=200,       # T/m/s
    slew_unit='T/m/s',
    rf_ringdown_time=30e-6,
    rf_dead_time=100e-6,
    adc_dead_time=10e-6
)

seq = pp.Sequence(system)

# --- Anatomy-dependent parameters ---
anatomy = "knee"  # change to "brain"

if anatomy == "brain":
    fov = 0.22
    slice_thickness = 5e-3
    matrix = 256
    tr = 4000e-3
    te = 90e-3
else:  # knee / ankle
    fov = 0.16
    slice_thickness = 3e-3
    matrix = 320
    tr = 3500e-3
    te = 70e-3

# --- RF pulse ---
rf, gz, _ = pp.make_sinc_pulse(
    flip_angle=np.pi/2,
    duration=3e-3,
    slice_thickness=slice_thickness,
    apodization=0.5,
    time_bw_product=4,
    system=system,
    return_gz=True
)

# --- Readout ---
gx = pp.make_trapezoid(
    channel='x',
    flat_area=matrix,
    flat_time=4e-3,
    system=system
)

adc = pp.make_adc(
    num_samples=matrix,
    duration=gx.flat_time,
    delay=gx.rise_time,
    system=system
)

# --- Sequence block ---
seq.add_block(rf, gz)
seq.add_block(gx, adc)

# --- Timing check ---
ok, error = seq.check_timing()
assert ok, error

# --- Export ---
seq.write(f"{anatomy}_tse_like.seq")

