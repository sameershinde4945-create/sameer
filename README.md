Siemens MRI software is structured
Author - Sameer Shinde 
<br>
<br>
import numpy as np
import pypulseq as pp


# ==========================================================
# 1. SYSTEM LIMITS (scanner capabilities)
# ==========================================================
system = pp.Opts(
    max_grad=80,            # mT/m
    grad_unit='mT/m',
    max_slew=200,           # T/m/s
    slew_unit='T/m/s',
    rf_ringdown_time=30e-6,
    rf_dead_time=100e-6,
    adc_dead_time=10e-6
)


# ==========================================================
# 2. PROTOCOL PARAMETERS (anatomy-specific)
# ==========================================================
def get_protocol(anatomy: str):
    """
    Returns protocol parameters based on anatomy.
    Same sequence engine, different configuration.
    """
    if anatomy.lower() == "brain":
        return {
            "fov": 0.22,
            "matrix": 256,
            "slice_thickness": 5e-3,
            "tr": 4.0,
            "te": 0.09
        }
    elif anatomy.lower() in ["knee", "ankle"]:
        return {
            "fov": 0.16,
            "matrix": 320,
            "slice_thickness": 3e-3,
            "tr": 3.5,
            "te": 0.07
        }
    else:
        raise ValueError("Unsupported anatomy")


protocol = get_protocol("knee")


# ==========================================================
# 3. SEQUENCE OBJECT
# ==========================================================
seq = pp.Sequence(system)


# ==========================================================
# 4. RF PULSES
# ==========================================================
rf90, gz90, _ = pp.make_sinc_pulse(
    flip_angle=np.pi / 2,
    duration=3e-3,
    slice_thickness=protocol["slice_thickness"],
    apodization=0.5,
    time_bw_product=4,
    system=system,
    return_gz=True
)

rf180, gz180, _ = pp.make_sinc_pulse(
    flip_angle=np.pi,
    duration=3e-3,
    slice_thickness=protocol["slice_thickness"],
    apodization=0.5,
    time_bw_product=4,
    system=system,
    return_gz=True
)


# ==========================================================
# 5. READOUT GRADIENT & ADC
# ==========================================================
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


# ==========================================================
# 6. TIMING CALCULATIONS
# ==========================================================
te_delay_1 = protocol["te"] / 2 - pp.calc_duration(rf90) / 2 - pp.calc_duration(gz90)
te_delay_2 = protocol["te"] / 2 - pp.calc_duration(rf180) / 2 - pp.calc_duration(gz180)

delay_te_1 = pp.make_delay(te_delay_1)
delay_te_2 = pp.make_delay(te_delay_2)

tr_delay = protocol["tr"] - (
    pp.calc_duration(rf90) +
    pp.calc_duration(delay_te_1) +
    pp.calc_duration(rf180) +
    pp.calc_duration(delay_te_2) +
    pp.calc_duration(gx)
)

delay_tr = pp.make_delay(tr_delay)


# ==========================================================
# 7. SEQUENCE BLOCK ASSEMBLY
# ==========================================================
seq.add_block(rf90, gz90)
seq.add_block(delay_te_1)

seq.add_block(rf180, gz180)
seq.add_block(delay_te_2)

seq.add_block(gx, adc)
seq.add_block(delay_tr)


# ==========================================================
# 8. VALIDATION & EXPORT
# ==========================================================
ok, timing_error = seq.check_timing()
if not ok:
    raise RuntimeError(timing_error)

seq.write("spin_echo_2d.seq")
