# SOLAR HIGH ALTITUDE LONG ENDURANCE AIRCRAFT

This paper is a sizing analysis done for a high altitude, long endurance solar aircraft.  The sizing was done using a convex optimization tool called, GPkit.  The optimization model was constructed using various models to describe the physics required to achieve the specified requirements. The specific requirements to which this plane was optimized are found in table 1. 

\begin{table}[H]
\begin{center}
\label{t:battery}
\begin{tabular}{ | c |  c | }
    \hline
    Requirement & Specification \\ \hline
    Payload & 10 [lbs] 10 [Watts] \\\hline
    Endurance & 5-40 days \\\hline
    Availability & 90\% \\\hline
    Latitude Coverage & 45-60 degrees \\\hline
\end{tabular}
\caption{Table outlining the requirements given to us by the customer.}
\end{center}
\end{table}


```python
#inPDF: skip

from gpkit import Variable, Model, units
from gpkit.constraints.set import ConstraintSet
from gpkit import LinkedConstraintSet
import numpy as np
import matplotlib.pyplot as plt
        
CD = Variable('C_D', '-', 'Drag coefficient')
CL = Variable('C_L', '-', 'Lift coefficient')
P_shaft = Variable('P_{shaft}', 'W', 'Shaft power')
S = Variable('S', 'm^2', 'Wing reference area')
V = Variable('V', 'm/s', 'Cruise velocity')
W = Variable('W', 'lbf', 'Aircraft weight')
rho = Variable(r'\rho', 'kg/m^3', 'Density of air')
E_batt = Variable('E_{batt}', 'J', 'Battery energy')
h = Variable("h", "ft", "Altitude")
g = Variable('g', 9.81, 'm/s^2', 'Gravitational acceleration')

```
# Steady Level Flight

We are assuming steady level flight. Specifically, the required lift must be equal to the weight.   Additionally, the shaft power produced by the engine has to be greater than the required flight power. 

```python
#inPDF: replace with SteadyLevelFlight.vars.generated.tex

class SteadyLevelFlight(Model):
    def __init__(self, **kwargs):
        eta_prop = Variable(r'\eta_{prop}', 0.7, '-', 'Propulsive efficiency')
```

```python
#inPDF: replace with SteadyLevelFlight.cnstrs.generated.tex

        constraints = [P_shaft >= V*W*CD/CL/eta_prop,   # eta*P = D*V
                       W == 0.5*rho*V**2*CL*S]
        Model.__init__(self, None, constraints, **kwargs)
```
# Aerodynamics 

We assumed that it the drag was combination of non-lifting drag, wing profile drag, and induced drag. 

```python
#inPDF: replace with Aero.vars.generated.tex

class Aero(Model):
    def __init__(self, **kwargs):

        Cd0 = Variable('C_{d0}', 0.002, '-', "non-wing drag coefficient")
        cdp = Variable("c_{dp}", "-", "wing profile drag coeff")
        CLmax = Variable('C_{L-max}', 1.5, '-', 'maximum lift coefficient')
        e = Variable('e', 0.9, '-', "spanwise efficiency")
        AR = Variable('AR', 27, '-', "aspect ratio")
        b = Variable('b', 'ft', 'span')
        mu = Variable(r'\mu', 1.5e-5, 'N*s/m^2', "dynamic viscosity")
        Re = Variable("Re", '-', "Reynolds number")
        Re_ref = Variable("Re_{ref}", 3e5, "-", "Reference Re for cdp")
        Cf = Variable("C_f", "-", "wing skin friction coefficient")
        Kwing = Variable("K_{wing}", 1.3, "-", "wing form factor")
```

```python
#inPDF: replace with Aero.cnstrs.generated.tex

        constraints = [
            CD >= (Cd0 + cdp + CL**2/(np.pi*e*AR))*1.3,
            cdp >= ((0.006 + 0.005*CL**2 + 0.00012*CL**10)*(Re/Re_ref)**-0.3),
            b**2 == S*AR,
            CL <= CLmax,
            Re == rho*V/mu*(S/AR)**0.5,
            ]
        Model.__init__(self, None, constraints, **kwargs)
```
# Weights

We assumed that the weight of the airframe, which includes the spar, wing, engines, and fuselage structural weight, is fixed at 20% of the weight of the aircraft. The energy battery density is based on the best available lithium ion battery.  The avionics weight is based on required avionics for flight control, ground communication, and satellite communication.  The solar cell area density is based off of the best available solar cell density. 



```python
#inPDF: replace with Weight.vars.generated.tex

class Weight(Model):
    def __init__(self, **kwargs):

        W_batt = Variable('W_{batt}', 'lbf', 'Battery weight')
        W_airframe = Variable('W_{airframe}', 'lbf', 'Airframe weight')
        W_solar = Variable('W_{solar}', 'lbf', 'Solar panel weight')
        W_pay = Variable('W_{pay}', 4, 'lbf', 'Payload weight')
        W_avionics = Variable('W_{avionics}', 4, 'lbf', 'avionics weight')
        rho_solar = Variable(r'\rho_{solar}', 1.2, 'kg/m^2',
                             'Solar cell area density')
        f_airframe = Variable('f_{airframe}', 0.20, '-',
                              'Airframe weight fraction')
        h_batt = Variable('h_{batt}', 250, 'W*hr/kg', 'Battery energy density')
```

```python
#inPDF: replace with Weight.cnstrs.generated.tex

        constraints = [W_airframe >= W*f_airframe,
                       W_batt >= E_batt/h_batt*g,
                       W_solar >= rho_solar*g*S,
                       W >= W_pay + W_solar + W_airframe + W_batt + W_avionics]
        Model.__init__(self, None, constraints, **kwargs)
```

# Power 

The value assumed for the average daytime solar irradiance, was based on standard solar irradiance values.   The accessory power draw is based on the approximate power usage from the avionics (15 Watts) and payload (10 Watts).  The night span is 8 hours and is based on the worst case scenario for the 45 degree latitude at the winter solstice. The assumption is that this is the worst case scenario for a solar aircraft. Successful flight on this day will guarantee flight on any other day of the year. The incidence angle is the angle of the sun shining on the solar array. The cosine of the angle determines the amount of solar irradiance reaching the array. Because the cosine function is not convex, the value for the 45th parallel at solar noon on the winter solstice was used as a constant. Solar cell efficiency is based on the best available technology. 

```python
#inPDF: replace with Power.vars.generated.tex

class Power(Model):
    def __init__(self, **kwargs):

        PS_irr = Variable('(P/S)_{irr}', 1000*0.5, 'W/m^2',
                          'Average daytime solar irradiance')
        P_oper = Variable('P_{oper}', 'W', 'Aircraft operating power')
        P_charge = Variable('P_{charge}', 'W', 'Battery charging power')
        P_acc = Variable('P_{acc}', 25, 'W', 'Accessory power draw')
        eta_solar = Variable(r'\eta_{solar}', 0.2, '-',
                             'Solar cell efficiency')
        eta_charge = Variable(r'\eta_{charge}', 0.95, '-',
                              'Battery charging efficiency')
        eta_discharge = Variable(r'\eta_{discharge}', 0.95, '-',
                                 'Battery discharging efficiency')
        t_day = Variable('t_{day}', 'hr', 'Daylight span')
        t_night = Variable('t_{night}', 16, 'hr', 'Night span')
        th = Variable(r'\theta', 0.35, '-', 
                      'cosine of incidence angle at 45 lat')

```

```python
#inPDF: replace with Power.cnstrs.generated.tex

        constraints = [PS_irr*eta_solar*S >= P_oper + P_charge,
                       P_oper >= P_shaft + P_acc,
                       P_charge >= E_batt/(t_day*eta_charge*th),
                       t_day + t_night <= 24*units.hr,
                       E_batt >= P_oper*t_night/eta_discharge]
        Model.__init__(self, None, constraints, **kwargs)
```

# Atmosphere 

Temperature, pressure and air density variations with altitude are based off on the standard atmosphere.

```python
#inPDF: replace with Atmosphere.vars.generated.tex

class Atmosphere(Model):
    def __init__(self, **kwargs):

        p_sl = Variable("p_{sl}", 101325, "Pa", "Pressure at sea level")
        T_sl = Variable("T_{sl}", 288.15, "K", "Temperature at sea level")
        L_atm = Variable("L_{atm}", 0.0065, "K/m", "Temperature lapse rate")
        T_atm = Variable("T_{atm}", "K", "air temperature")
        M_atm = Variable("M_{atm}", 0.0289644, "kg/mol",
                         "Molar mass of dry air")
        R_atm = Variable("R_{atm}", 8.31447, "J/mol/K", "air specific heating value")
        TH = (g*M_atm/R_atm/L_atm).value
```

```python
#inPDF: replace with Atmosphere.cnstrs.generated.tex

        constraints = [
            #h <= 20000*units.m,  # Model valid to top of troposphere
            T_sl >= T_atm + L_atm*h,     # Temp decreases w/ altitude
            # http://en.wikipedia.org/wiki/Density_of_air#Altitude
            rho <= p_sl*T_atm**(TH-1)*M_atm/R_atm/(T_sl**TH)]
        Model.__init__(self, None, constraints, **kwargs)
```

# Station Keeping Requirements

To achieve the minimum footprint of 100km, the aircraft must fly at 15,000 ft. In order to avoid diminished solar irradiance due to weather, and to avoid air traffic conflicts, the aircraft must fly at 50,000 ft or above. 

\begin{figure}[h!]
	\label{f:windspeeds}
	\begin{center}
	\includegraphics[scale = 0.45]{windspeeds}
	\caption{This figure shows the wind speeds necessary to station keep at the 90\%, 95\% and the 99\% winds for a given altitude.  As can be seen, the winds at 50,000 ft are higher than those at 15,000 ft.} 
	\end{center}
\end{figure}

Please note that the wind speed in this model was set to 10 m/s and the minimum altitude to 15,000 ft to ensure the feasibility of the optimization.  After the initial sizing, both the wind speed and minimum altitude were varied to observe its effect on the size of the aircraft.  The results of that study are shown below. 

```python
#inPDF: replace with StationKeeping.vars.generated.tex
class StationKeeping(Model):
    def __init__(self, **kwargs):
        
        h_min = Variable('h_{min}', 15000, 'ft', 'minimum altitude')
        V_wind = Variable('V_{wind}', 10, 'm/s', 'wind speed')
        
```

```python
#inPDF: replace with StationKeeping.cnstrs.generated.tex
        constraints = [h >= h_min,
                       V >= V_wind]
        Model.__init__(self, None, constraints, **kwargs)
```

# Objective

The objective function of the optimzation was set to minimize the aircraft weight. Because our customer expressed affordability as a factor in our design we chose to minimize weight since cost scales primarily with weight. The "cost" shown is the optimized weight of the aircraft for the given constraints. 

```python
#inPDF: skip

class SolarHALE(Model):
    """High altitude long endurance solar UAV"""
    def __init__(self, **kwargs):
        """Setup method should return objective, list of constraints"""

        slf = SteadyLevelFlight()
        power = Power()
        weight = Weight()
        sk = StationKeeping()
        aero = Aero()
        atmosphere = Atmosphere()
        self.submodels = [slf, power, weight, sk, aero, atmosphere] 



        constraints = []
        lc = LinkedConstraintSet([self.submodels, constraints])

        objective = W

        Model.__init__(self, objective, lc, **kwargs)
```

```python
#inPDF: replace with sol.generated.tex


if __name__ == "__main__":
    M = SolarHALE()
    sol = M.solve("mosek")
    from gpkit.small_scripts import unitstr

    def gen_model_tex(model, modelname):
        with open('%s.vars.generated.tex' % modelname, 'w') as f:
            f.write("\\begin{longtable}{llll}\n \\toprule\n")
            f.write("\\toprule\n")
            f.write("Variables & Value & Units & Description \\\\ \n")
            f.write("\\midrule\n")
            #f.write("\\multicolumn{3}{l}\n")
            for var in model.varkeys:
                unitstr = var.unitstr()[1:]
                unitstr = "$[%s]$" % unitstr if unitstr else ""
                val = "%0.3f" % var.value if var.value else ""
                f.write("$%s$ & %s & %s & %s \\\\\n" % (var.name, val, unitstr, var.label))
            f.write("\\bottomrule\n")
            f.write("\\end{longtable}\n")

        with open('%s.cnstrs.generated.tex' % modelname, 'w') as f:
            lines = model.latex(excluded=["models"]).replace("[ll]", "{ll}").split("\n")
            modeltex = "\n".join(lines[:1] + lines[3:])
            f.write("$$ %s $$" % modeltex)

    for model in M.submodels:
        gen_model_tex(model, model.__class__.__name__)
    with open("sol.generated.tex", "w") as f:
        f.write(sol.table(latex=True))

    # number of data points
    #N = 30

    #M.substitutions.update({ 'h_{batt}': ('sweep', [250,350,500])})

    ## plotting at 45 degree latitude

    #M.substitutions.update({'V_{wind}': ('sweep', np.linspace(5,40,N))})
    #sol = M.solve(solver='mosek', verbosity=0, skipsweepfailures=True)
    #
    #W = sol('W')
    #b = sol('b')
    #V_wind = sol('V_{wind}')
    #ind = np.nonzero(V_wind.magnitude==5.)
    #ind = ind[0]

    #plt.close()
    #plt.plot(V_wind[ind[0]:ind[1]], b[ind[0]:ind[1]], V_wind[ind[1]:ind[2]], b[ind[1]:ind[2]], V_wind[ind[2]:V_wind.size],  b[ind[2]:V_wind.size])
    #plt.ylabel('wing span [ft]')
    #plt.xlabel('wind speed [m/s]')
    #plt.legend(['h_batt = 250', 'h_batt = 350', 'h_batt = 500']), 
    #plt.grid()
    #plt.axis([5,40,0,200])
    #plt.savefig('bvsV_wind.png')
    #
    #M.substitutions.update({'V_{wind}': 10})
    #M.substitutions.update({'h': ('sweep', np.linspace(15000,50000,N))})
    #sol = M.solve(solver='mosek', verbosity=0, skipsweepfailures=True)
    #
    #W = sol('W')
    #b = sol('b')
    #h = sol('h')
    #ind = np.nonzero(h.magnitude==15000.)
    #ind = ind[0]

    #plt.close()
    #plt.plot(h[ind[0]:ind[1]], b[ind[0]:ind[1]], h[ind[1]:ind[2]], b[ind[1]:ind[2]], h[ind[2]:h.size],  b[ind[2]:h.size])
    #plt.ylabel('wing span [ft]')
    #plt.xlabel('altitude [ft]')
    #plt.legend(['h_batt = 250', 'h_batt = 350', 'h_batt = 500']), 
    #plt.grid()
    #plt.axis([15000,50000,0,200])
    #plt.savefig('bvsh.png')
    #
    ## plotting at 30 degree latitude
    #M.substitutions.update({r'\theta': 0.6, 't_{night}': 14})

    #M.substitutions.update({'h':15000})
    #M.substitutions.update({'V_{wind}': ('sweep', np.linspace(5,40,N))})
    #sol = M.solve(solver='mosek', verbosity=0, skipsweepfailures=True)
    #
    #W = sol('W')
    #b = sol('b')
    #V_wind = sol('V_{wind}')
    #ind = np.nonzero(V_wind.magnitude==5.)
    #ind = ind[0]

    #plt.close()
    #plt.plot(V_wind[ind[0]:ind[1]], b[ind[0]:ind[1]], V_wind[ind[1]:ind[2]], b[ind[1]:ind[2]], V_wind[ind[2]:V_wind.size],  b[ind[2]:V_wind.size])
    #plt.ylabel('wing span [ft]')
    #plt.xlabel('wind speed [m/s]')
    #plt.legend(['h_batt = 250', 'h_batt = 350', 'h_batt = 500']), 
    #plt.grid()
    #plt.axis([5,40,0,200])
    #plt.savefig('bvsV_wind30.png')
    #
    #M.substitutions.update({'V_{wind}': 10})
    #M.substitutions.update({'h': ('sweep', np.linspace(15000,50000,N))})
    #sol = M.solve(solver='mosek', verbosity=0, skipsweepfailures=True)
    #
    #W = sol('W')
    #b = sol('b')
    #h = sol('h')
    #ind = np.nonzero(h.magnitude==15000.)
    #ind = ind[0]

    #plt.close()
    #plt.plot(h[ind[0]:ind[1]], b[ind[0]:ind[1]], h[ind[1]:ind[2]], b[ind[1]:ind[2]], h[ind[2]:h.size],  b[ind[2]:h.size])
    #plt.ylabel('wing span [ft]')
    #plt.xlabel('altitude [ft]')
    #plt.legend(['h_batt = 250', 'h_batt = 350', 'h_batt = 500']), 
    #plt.grid()
    #plt.axis([15000,50000,0,200])
    #plt.savefig('bvsh30.png')
```

# Results

After an initial sizing was done we wanted to vary a few of values to understand how the design changed as we approached the requirements.  Each graph below was plotted for 3 battery cases shown in table 2.

\begin{table}[H]
\begin{center}
\label{t:battery}
\begin{tabular}{ | c |  c | }
    \hline
    Battery Efficiency & Battery Type \\ \hline
    250 $[Whr/kg]$ & Best known lithium ion battery \\\hline
    350 $[Whr/kg]$ & Best known lithium sulphur battery \\
    & (used on the Zephyr) \\\hline
    500 $[Whr/kg]$ & Theoretical limit \\
    & (only achieved in labs) \\\hline
\end{tabular}
\caption{Table showing the best battery efficiencies for lithium ion, lithium sulphur and the theoretical limit.}
\end{center}
\end{table}

# Size vs Wind Speed

As explained in figure 1, wind speeds are important for this analysis because the wind affects the size of the engine and therefore battery needed to sustain flight. As shown in figure 2, in order to reach the 90th percentile winds (i.e. 90% availability), the wing span must be in excess of 200 ft.  Please note that this result was done assuming that we are flying at 45 degrees latitude and during the winter solstice at 15,000 ft. These conditions will guarantee year-round availability.  Figure 3 is shows the same plot but for 30 degree latitude. 


\begin{figure}[H]
	\label{f:bvsV_wind}
	\begin{center}
	\includegraphics[scale = 0.45]{bvsV_wind}
	\caption{This figure shows how at the 45 degree latitude, at the winter solstice, in order to overcome the 90\% windspeeds at 15,000 ft the aircraft needs to be larger than is structurally feasible.}
	\end{center}
\end{figure}

\begin{figure}[H]
	\label{f:bvsV_wind30}
	\begin{center}
	\includegraphics[scale = 0.45]{bvsV_wind30}
	\caption{This figure shows how at the 30 degree latitude, at the winter solstice, in order to overcome the 90\% wind speeds at 15,000 ft the aircraft needs to be larger than is structurally feasible.}
	\end{center}
\end{figure}

# Size vs Altitude

Reaching an altitude of near 50,000 ft is desirable to: 1) reach lower winds, and 2) escape heavy jet stream.  The difficulty with going higher is mainly that the air is thinner and requires faster speeds to produce sufficient lift.  Faster speeds mean larger engines, which means larger batteries and solar cell area.  Figure 4 shows the trade off in size, shown in wing span, vs altitude.  To reach an altitude of 50,000 ft or higher the plane must become larger than is feasibly buildable. Figure 5 shows the same trade off but for the 30 degree latitude case. Please note that this study assumes no wind.  So even if it were possible to reach higher altitudes, the higher winds at that altitude would still make this design infeasible. 

\begin{figure}[H]
	\label{f:bvsh}
	\begin{center}
	\includegraphics[scale = 0.45]{bvsh}
	\caption{This figure shows how at the 45 degree latitude, during the winter solstice, in order to reach altitudes of 50,000 ft the plane must be larger than is structurally feasible. (Note: this study assumes no wind)}
	\end{center}
\end{figure}

\begin{figure}[H]
	\label{f:bvsh30}
	\begin{center}
	\includegraphics[scale = 0.45]{bvsh30}
	\caption{This figure shows how at the 30 degree latitude, during the winter solstice, in order to reach altitudes of 50,000 ft the plane must be larger than is structurally feasible. (Note: this study assumes no wind)}
	\end{center}
\end{figure}

# Conclusion

Based on the sizing results done in this study we believe that a solar aircraft cannot achieve the threshold requirements outlined by the customer.  Flying at the 45 parallel on the winter solstice gives insufficient sunlight at low angles to the solar array. Combined with the need to fly in windy conditions, the aircraft size grows to unbuildable dimensions. If the customer is willing to sacrifice availability by flying in slower winds at lower latitudes and limited periods of the year, a solar option could be further explored.   