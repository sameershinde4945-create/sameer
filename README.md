Siemens MRI software is structured
Author - Sameer Shinde 
import numpy as np
import pypulseq as pp

# System limits (scanner abstraction)
system = pp.Opts(
    max_grad=80,
    grad_unit='mT/m',
    max_slew=200,
    slew_unit='T/m/s',
    rf_ringdown_time=30e-6,
    rf_dead_time=100e-6,
    adc_dead_time=10e-6
)

# Anatomy-based protocol parameters
def get_protocol(anatomy):
    if anatomy == "brain":
        return dict(fov=0.22, matrix=256, slice=5e-3, tr=4.0, te=0.09)
    elif anatomy in ["knee", "ankle"]:
        return dict(fov=0.16, matrix=320, slice=3e-3, tr=3.5, te=0.07)
    else:
        raise ValueError("Unsupported anatomy")

protocol = get_protocol("knee")

# Sequence object
seq = pp.Sequence(system)

# RF pulses
rf90, gz90, _ = pp.make_sinc_pulse(
    flip_angle=np.pi/2,
    duration=3e-3,
    slice_thickness=protocol["slice"],
    apodization=0.5,
    time_bw_product=4,
    system=system,
    return_gz=True
)

rf180, gz180, _ = pp.make_sinc_pulse(
    flip_angle=np.pi,
    duration=3e-3,
    slice_thickness=protocol["slice"],
    apodization=0.5,
    time_bw_product=4,
    system=system,
    return_gz=True
)

# Readout gradient and ADC
delta_k = 1 / protocol["fov"]
gx = pp.make_trapezoid(
    channel='x',
    flat_area=protocol["matrix"] * delta_k,
    flat_time=4e-3,
    system=system
)

adc = pp.make_adc(
    num_samples=protocol["matrix"],
    duration=gx.flat_time,
    delay=gx.rise_time,
    system=system
)

# Simple timing
seq.add_block(rf90, gz90)
seq.add_block(pp.make_delay(protocol["te"] / 2))
seq.add_block(rf180, gz180)
seq.add_block(pp.make_delay(protocol["te"] / 2))
seq.add_block(gx, adc)
seq.add_block(pp.make_delay(protocol["tr"]))

# Validation and export
ok, error = seq.check_timing()
if not ok:
    raise RuntimeError(error)

seq.write("spin_echo_2d.seq")
