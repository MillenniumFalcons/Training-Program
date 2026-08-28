# Tips for an Easier Time in Build Season

## Backlash
- when the mechanism can move without the motor moving
- leads to very shitty and sloppy controls
- fixed by good chain/belt tensioning
The best way to tension a chain is through a turnbuckle, which requires a long chain run for all pivoting mechanisms. **Remember to advocate for yourself, and ask for long chain runs on chain driven pivots**

## Planetaries
- premade planetary gearboxes (maxPlanetary/versaPlanetary) are kinda shit for position controlled mechanisms like pivots or arms
- try to get designers to use custom gearboxes for position controlled mechs (they probly alr know this)

## Pivot Weight
- a heavy weight (like a motor or metal plates) at the end of a long arm/pivot are bad
- they lead to sloppy and hard-to-control movement
- correct designers if the end plates on arms are too heavy / made of aluminum, and see if roller motors can be moved off the pivot, or close to the pivoting point
	- again, designers probly alr know this

## Vision
- camera mounts should be at oblique angles to tags
- plan with design to get cameras at good spots early in the design process

## Sim everything
- make your entire robot in sim with mechanism2ds as early as possible, even before design is complete
- start simming shit as soon as you know what the superstructure will look like
- **The only thing you should have to change in your code once the robot is built are the PID/conversion constants, everything else should be perfect in sim** 
