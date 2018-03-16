## TAEP: Traffic Analysis and Experimentation Platform

TAEP is a Traffic Analysis and Experimentation platform based on the Barefoot/Tofino hardware. It provides a safe and reliable way to run A/B experiments in a production network.

### Sub-Projects

#### TAEP-Scripts
[github.com/att-innovate/taep-scripts](https://github.com/att-innovate/taep-scripts) contains all the shell scripts to build and manage TAEP.

#### TAEP-Controller
[github.com/att-innovate/taep-controller](https://github.com/att-innovate/taep-controller) contains all the code for the control-plane. The platform controller itself is written in [Rust](https://www.rust-lang.org/en-US/), and the data-plane related code is written in [P4](https://p4.org).

#### TAEP-Analytics
[github.com/att-innovate/taep-analytics](https://github.com/att-innovate/taep-analytics) contains the components related to the analytics stack.

### Sample Experiments
[TAEP-Examples](https://github.com/att-innovate/taep/blob/master/EXAMPLES.md) lists some sample experiments that can be run on TAEP.

### Installation
The TAEP code, the scripts, and experiments got developed and tested on following Barefoot/Tofino based setup:

- Hardware
	- Edgecore Wedge 100BF-32X
	- Edgecore Wedge 100BF-65X
- Software
	- ONL-2.0.0 2017-08-02.0609
	- bf-sde-7.0.0.18 

Assuming you have the same setup just follow the installation steps as outlined in the [TAEP-Scripts](https://github.com/att-innovate/taep-scripts) sub-project.

