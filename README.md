# fleet - a distributed init system

[![Build Status](https://travis-ci.org/coreos/fleet.png?branch=master)](https://travis-ci.org/coreos/fleet)

fleet ties together [systemd](http://coreos.com/using-coreos/systemd) and [etcd](https://github.com/coreos/etcd) into a simple distributed init system. Think of it as an extension of systemd that operates at the cluster level instead of the machine level. **This project is very low level and is designed as a foundation for higher order orchestration.**

fleet is oriented around systemd units and is not a container manager or orchestration system. fleet supports very basic scheduling of systemd units in a cluster. Those looking for more complex scheduling requirements or a first-class container orchestration system should check out [Kubernetes](https://kubernetes.io).

## Using fleet

Launching a unit with fleet is as simple as running `fleetctl start`:

```
$ fleetctl start examples/hello.service
Unit hello.service launched on 113f16a7.../172.17.8.103
```

The `fleetctl start` command waits for the unit to get scheduled and actually start somewhere in the cluster.
`fleetctl list-unit-files` tells you the desired state of your units and where they are currently scheduled:

```
$ fleetctl list-unit-files
UNIT            HASH     DSTATE    STATE     TMACHINE
hello.service   e55c0ae  launched  launched  113f16a7.../172.17.8.103
```

`fleetctl list-units` exposes the systemd state for each unit in your fleet cluster:

```
$ fleetctl list-units
UNIT            MACHINE                    ACTIVE   SUB
hello.service   113f16a7.../172.17.8.103   active   running
```

## Supported Deployment Patterns

* Deploy a single unit anywhere on the cluster
* Deploy a unit globally everywhere in the cluster
* Automatic rescheduling of units on machine failure
* Ensure that units are deployed together on the same machine
* Forbid specific units from colocation on the same machine (anti-affinity)
* Deploy units to machines only with specific metadata

These patterns are all defined using [custom systemd unit options][unit-files].

[unit-files]: https://github.com/coreos/fleet/blob/master/Documentation/unit-files-and-scheduling.md#fleet-specific-options

## Getting Started

Before you can deploy units, fleet must be [deployed and configured][deploy-and-configure] on each host in your cluster. (If you are running CoreOS, fleet is already installed.)

After you have machines configured (check `fleetctl list-machines`), get to work with the [client][using-the-client.md].

[using-the-client.md]: https://github.com/coreos/fleet/blob/master/Documentation/using-the-client.md
[deploy-and-configure]: https://github.com/coreos/fleet/blob/master/Documentation/deployment-and-configuration.md

### Building

fleet must be built with Go 1.4+ on a Linux machine. Simply run `./build` and then copy the binaries out of bin/ onto each of your machines. The tests can similarly be run by simply invoking `./test`.

If you're on a machine without Go 1.4+ but you have Docker installed, run `./build-docker` to compile the binaries instead.

## Project Details

### API

The fleet API uses JSON over HTTP to manage units in a fleet cluster.
See the [API documentation][api-doc] for more information.

[api-doc]: https://github.com/coreos/fleet/blob/master/Documentation/api-v1.md

### Release Notes

See the [releases tab](https://github.com/coreos/fleet/releases) for more information on each release.

### Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for details on submitting patches and contacting developers via IRC and mailing lists.

### License

fleet is released under the Apache 2.0 license. See the [LICENSE](LICENSE) file for details.

Specific components of fleet use code derivative from software distributed under other licenses; in those cases the appropriate licenses are stipulated alongside the code.



########################## REDUCE_ETCD_REQUEST ################## 804 1257


diff --git a/agent/reconcile.go b/agent/reconcile.go
index ef092cb..0a684b6 100644
--- a/agent/reconcile.go
+++ b/agent/reconcile.go
@@ -115,15 +115,9 @@ func (ar *AgentReconciler) Purge(a *Agent) {
 // desiredAgentState builds an *AgentState object that represents what the
 // provided Agent should currently be doing.
 func desiredAgentState(a *Agent, reg registry.Registry) (*AgentState, error) {
-	units, err := reg.Units()
+	units, sUnits, err:= reg.UnitsAndSchedule()
 	if err != nil {
-		log.Errorf("Failed fetching Units from Registry: %v", err)
-		return nil, err
-	}
-
-	sUnits, err := reg.Schedule()
-	if err != nil {
-		log.Errorf("Failed fetching schedule from Registry: %v", err)
+		log.Errorf("Failed fetching Units or schedule from Registry: %v", err)
 		return nil, err
 	}
 
diff --git a/engine/engine.go b/engine/engine.go
index bce3ebe..7a32383 100644
--- a/engine/engine.go
+++ b/engine/engine.go
@@ -227,15 +227,9 @@ func (e *Engine) Trigger() {
 }
 
 func (e *Engine) clusterState() (*clusterState, error) {
-	units, err := e.registry.Units()
+	units, sUnits, err := e.registry.UnitsAndSchedule()
 	if err != nil {
-		log.Errorf("Failed fetching Units from Registry: %v", err)
-		return nil, err
-	}
-
-	sUnits, err := e.registry.Schedule()
-	if err != nil {
-		log.Errorf("Failed fetching schedule from Registry: %v", err)
+		log.Errorf("Failed fetching Units or schedule from Registry: %v", err)
 		return nil, err
 	}
 
diff --git a/registry/fake.go b/registry/fake.go
index 2a6c8bc..2812cce 100644
--- a/registry/fake.go
+++ b/registry/fake.go
@@ -88,6 +88,41 @@ func (f *FakeRegistry) Machines() ([]machine.MachineState, error) {
 	return f.machines, nil
 }
 
+func (f *FakeRegistry) UnitsAndSchedule() ([]job.Unit, []job.ScheduledUnit, error) {
+	f.RLock()
+	defer f.RUnlock()
+
+	var sorted sort.StringSlice
+	for _, j := range f.jobs {
+		sorted = append(sorted, j.Name)
+	}
+	sorted.Sort()
+
+	units := make([]job.Unit, len(f.jobs))
+	for i, jName := range sorted {
+		j := f.jobs[jName]
+		u := job.Unit{
+			Name:        j.Name,
+			Unit:        j.Unit,
+			TargetState: j.TargetState,
+		}
+		units[i] = u
+	}
+
+	sUnits := make([]job.ScheduledUnit, 0, len(f.jobs))
+	for _, jName := range sorted {
+		j := f.jobs[jName]
+		su := job.ScheduledUnit{
+			Name:            j.Name,
+			State:           j.State,
+			TargetMachineID: j.TargetMachineID,
+		}
+		sUnits = append(sUnits, su)
+	}
+
+	return units, sUnits, nil
+}
+
 func (f *FakeRegistry) Units() ([]job.Unit, error) {
 	f.RLock()
 	defer f.RUnlock()
diff --git a/registry/interface.go b/registry/interface.go
index 270a2ef..6b90b13 100644
--- a/registry/interface.go
+++ b/registry/interface.go
@@ -47,6 +47,7 @@ type UnitRegistry interface {
 	Unit(name string) (*job.Unit, error)
 	Units() ([]job.Unit, error)
 	UnitStates() ([]*unit.UnitState, error)
+	UnitsAndSchedule() ([]job.Unit, []job.ScheduledUnit, error)
 }
 
 type ClusterRegistry interface {
diff --git a/registry/job.go b/registry/job.go
index 2b11184..2a39e17 100644
--- a/registry/job.go
+++ b/registry/job.go
@@ -31,8 +31,7 @@ const (
 	jobPrefix = "job"
 )
 
-// Schedule returns all ScheduledUnits known by fleet, ordered by name
-func (r *EtcdRegistry) Schedule() ([]job.ScheduledUnit, error) {
+func (r *EtcdRegistry) registry() (*etcd.Response, error) {
 	key := r.prefixed(jobPrefix)
 	opts := &etcd.GetOptions{
 		Sort:      true,
@@ -43,9 +42,40 @@ func (r *EtcdRegistry) Schedule() ([]job.ScheduledUnit, error) {
 		if isEtcdError(err, etcd.ErrorCodeKeyNotFound) {
 			err = nil
 		}
-		return nil, err
 	}
+	return res, err
+}
 
+func (r *EtcdRegistry) units(res *etcd.Response) ([]job.Unit, error) {
+	uMap := make(map[string]*job.Unit)
+	for _, dir := range res.Node.Nodes {
+		u, err := r.dirToUnit(dir)
+		if err != nil {
+			log.Errorf("Failed to parse Unit from etcd: %v", err)
+			continue
+		}
+		if u == nil {
+			continue
+		}
+		uMap[u.Name] = u
+	}
+
+	var sortable sort.StringSlice
+	for name, _ := range uMap {
+		sortable = append(sortable, name)
+	}
+	sortable.Sort()
+
+	units := make([]job.Unit, 0, len(sortable))
+	for _, name := range sortable {
+		units = append(units, *uMap[name])
+	}
+
+	return units, nil
+
+}
+
+func (r *EtcdRegistry) schedule(res *etcd.Response) ([]job.ScheduledUnit, error) {
 	heartbeats := make(map[string]string)
 	uMap := make(map[string]*job.ScheduledUnit)
 
@@ -84,48 +114,43 @@ func (r *EtcdRegistry) Schedule() ([]job.ScheduledUnit, error) {
 		units = append(units, *uMap[name])
 	}
 	return units, nil
+
 }
 
-// Units lists all Units stored in the Registry, ordered by name. This includes both global and non-global units.
-func (r *EtcdRegistry) Units() ([]job.Unit, error) {
-	key := r.prefixed(jobPrefix)
-	opts := &etcd.GetOptions{
-		Sort:      true,
-		Recursive: true,
+func (r *EtcdRegistry) UnitsAndSchedule() ([]job.Unit, []job.ScheduledUnit, error) {
+	res, err := r.registry();
+	if err != nil || res == nil {
+		return nil, nil, err
 	}
-	res, err := r.kAPI.Get(r.ctx(), key, opts)
+
+	units, err := r.units(res)
 	if err != nil {
-		if isEtcdError(err, etcd.ErrorCodeKeyNotFound) {
-			err = nil
-		}
-		return nil, err
+		return nil, nil, err
 	}
 
-	uMap := make(map[string]*job.Unit)
-	for _, dir := range res.Node.Nodes {
-		u, err := r.dirToUnit(dir)
-		if err != nil {
-			log.Errorf("Failed to parse Unit from etcd: %v", err)
-			continue
-		}
-		if u == nil {
-			continue
-		}
-		uMap[u.Name] = u
+	sunits, err := r.schedule(res)
+	if err != nil {
+		return nil, nil, err
 	}
 
-	var sortable sort.StringSlice
-	for name, _ := range uMap {
-		sortable = append(sortable, name)
+	return units, sunits, err
+}
+// Schedule returns all ScheduledUnits known by fleet, ordered by name
+func (r *EtcdRegistry) Schedule() ([]job.ScheduledUnit, error) {
+	res, err := r.registry();
+	if err != nil  || res == nil {
+		return nil, err
 	}
-	sortable.Sort()
+	return r.schedule(res)
+}
 
-	units := make([]job.Unit, 0, len(sortable))
-	for _, name := range sortable {
-		units = append(units, *uMap[name])
+// Units lists all Units stored in the Registry, ordered by name. This includes both global and non-global units.
+func (r *EtcdRegistry) Units() ([]job.Unit, error) {
+	res, err := r.registry();
+	if err != nil  || res == nil {
+		return nil, err
 	}
-
-	return units, nil
+	return r.units(res)
 }
 
 // Unit retrieves the Unit by the given name from the Registry. Returns nil if



########################## OVERWRITE ################## 760 614

commit cd5c2bef649b6e121e9b6aea8c21f2df502483eb
Author: h00229878 <huangshaoyu@huawei.com>
Date:   Thu Jul 2 18:54:25 2015 +0800

    enable overwrite unit(s)

diff --git a/fleetctl/fleetctl.go b/fleetctl/fleetctl.go
index 97057cd..cb0d17e 100644
--- a/fleetctl/fleetctl.go
+++ b/fleetctl/fleetctl.go
@@ -175,6 +175,7 @@ func init() {
 		cmdUnloadUnit,
 		cmdVerifyUnit,
 		cmdVersion,
+		cmdOverwriteUnit,
 	}
 }
 
@@ -664,13 +665,13 @@ func lazyCreateUnits(args []string) error {
 	return nil
 }
 
-func warnOnDifferentLocalUnit(loc string, su *schema.Unit) {
+func warnOnDifferentLocalUnit(loc string, su *schema.Unit) bool {
 	suf := schema.MapSchemaUnitOptionsToUnitFile(su.Options)
 	if _, err := os.Stat(loc); !os.IsNotExist(err) {
 		luf, err := getUnitFromFile(loc)
 		if err == nil && luf.Hash() != suf.Hash() {
 			stderr("WARNING: Unit %s in registry differs from local unit file %s", su.Name, loc)
-			return
+			return true
 		}
 	}
 	if uni := unit.NewUnitNameInfo(path.Base(loc)); uni != nil && uni.IsInstance() {
@@ -679,9 +680,11 @@ func warnOnDifferentLocalUnit(loc string, su *schema.Unit) {
 			tmpl, err := getUnitFromFile(file)
 			if err == nil && tmpl.Hash() != suf.Hash() {
 				stderr("WARNING: Unit %s in registry differs from local template unit file %s", su.Name, uni.Template)
+				return true
 			}
 		}
 	}
+	return false
 }
 
 func lazyLoadUnits(args []string) ([]*schema.Unit, error) {
diff --git a/fleetctl/overwrite.go b/fleetctl/overwrite.go
new file mode 100644
index 0000000..8d63a30
--- /dev/null
+++ b/fleetctl/overwrite.go
@@ -0,0 +1,117 @@
+package main
+
+import (
+	"os"
+
+	"github.com/coreos/fleet/job"
+)
+
+var cmdOverwriteUnit = &Command{
+	Name:    "overwrite",
+	Summary: "Overwrite one or more units in the cluster",
+	Usage:   "UNIT...",
+	Description: `Overwrite one or more running or submitted units from the cluster.
+
+Act as a procedure of stop-->destroy-->commit-->start(if needed) on unit(s)`,
+
+	Run: runOverwriteUnits,
+}
+
+
+
+func runOverwriteUnits(args []string) (exit int) {
+	for _, v := range args {
+		v = maybeAppendDefaultUnitType(v)
+		name := unitNameMangle(v)
+
+		u, err := cAPI.Unit(name)
+		if err != nil {
+			stderr("error retrieving Unit(%s) from Registry: %v", name, err)
+			return 1;
+		}
+		if u == nil {
+			stdout("Unit(%s) in can not be found in Registry, nothing to do with it", name)
+			continue
+		}
+
+		hash_mismatch := warnOnDifferentLocalUnit(v, u)
+		if hash_mismatch == false {
+			stdout("Nothing different between Unit(%s) in registory and local unit file, just skip", name)
+			continue
+		}
+
+		stopping := make([]string, 0)
+		stopping = append(stopping, u.Name)
+		if job.JobState(u.CurrentState) == job.JobStateLaunched || suToGlobal(*u) {
+			stdout("Stop Unit(%s) first, so that we can overwrite unit file", u.Name)
+			cAPI.SetUnitTargetState(u.Name, string(job.JobStateLoaded))
+			errchan := waitForUnitStates(stopping, job.JobStateLoaded, 0, os.Stdout)
+			for err := range errchan {
+				stderr("Error waiting for units: %v", err)
+				exit = 1
+			}
+		}
+
+		err = cAPI.DestroyUnit(name)
+		if err != nil {
+			continue
+		}
+
+		recreating := make([]string, 0)
+		recreating = append(recreating, v)
+		stdout("Recreating unit(%s)", v)
+		if err := lazyCreateUnits(recreating); err != nil {
+			stderr("Error creating units: %v", err)
+			return;
+		}
+		if job.JobState(u.CurrentState) == job.JobStateInactive {
+			continue;
+		} else if job.JobState(u.CurrentState) == job.JobStateLoaded {
+			stdout("Rescheduling unit(%s)", v)
+			triggered, err := lazyLoadUnits(recreating)
+			if err != nil {
+				stderr("Error loading units: %v", err)
+				return 1
+			}
+
+			var loading []string
+			for _, u := range triggered {
+				if suToGlobal(*u) {
+					stdout("Triggered global unit %s load", u.Name)
+				} else {
+					loading = append(loading, u.Name)
+				}
+			}
+
+			errchan := waitForUnitStates(loading, job.JobStateLoaded, sharedFlags.BlockAttempts, os.Stdout)
+			for err := range errchan {
+				stderr("Error waiting for units: %v", err)
+			}
+		} else if job.JobState(u.CurrentState) ==  job.JobStateLaunched{
+			stdout("Restarting unit(%s)", v)
+			triggered, err := lazyStartUnits(recreating)
+			if err != nil {
+				stderr("Error starting units: %v", err)
+				return 1
+			}
+
+			var starting []string
+			for _, u := range triggered {
+				if suToGlobal(*u) {
+				stdout("Triggered global unit %s start", u.Name)
+				} else {
+					starting = append(starting, u.Name)
+				}
+			}
+
+			errchan := waitForUnitStates(starting, job.JobStateLaunched, sharedFlags.BlockAttempts, os.Stdout)
+			for err := range errchan {
+				stderr("Error waiting for units: %v", err)
+				exit = 1
+			}
+		}
+
+	}
+	return
+}
+


########################## metadata support relationalops ################## 1143

diff --git a/job/job.go b/job/job.go
index b0a0400..10f0387 100644
--- a/job/job.go
+++ b/job/job.go
@@ -260,7 +260,17 @@ func (j *Job) RequiredTargetMetadata() map[string]pkg.Set {
 		fleetMachineMetadata,
 	} {
 		for _, valuePair := range j.requirements()[key] {
-			s := strings.Split(valuePair, "=")
+			var s []string
+			for _, sep := range []string{"<=", ">=", "!=", "<", ">"} {
+				index := strings.Index(valuePair, sep)
+				if index != -1 {
+					s = []string{valuePair[0:index], valuePair[index:]}
+					break
+				}
+			}
+			if s == nil {
+				s = strings.Split(valuePair, "=")
+			}
 
 			if len(s) != 2 {
 				continue
diff --git a/machine/machine.go b/machine/machine.go
index 8be43d5..3632235 100644
--- a/machine/machine.go
+++ b/machine/machine.go
@@ -15,6 +15,9 @@
 package machine
 
 import (
+	"strings"
+	"strconv"
+	
 	"github.com/coreos/fleet/log"
 	"github.com/coreos/fleet/pkg"
 )
@@ -38,8 +41,81 @@ func HasMetadata(state *MachineState, metadata map[string]pkg.Set) bool {
 		if values.Contains(local) {
 			log.Debugf("Local Metadata(%s) meets requirement", key)
 		} else {
-			log.Debugf("Local Metadata(%s) does not match requirement", key)
-			return false
+			vs := values.Values()
+			for _, v := range vs {
+				if index := strings.Index(v, "<="); strings.Contains(v, "<=") && (index == 0) {
+					need, err1 := strconv.Atoi(v[2:])
+					have, err2 := strconv.Atoi(local)
+					if (err1 == nil && err2 == nil) {
+						if have <= need {
+							log.Debugf("Local Metadata(%s) meets requirement", key)
+							continue
+						} else {
+							log.Debugf("Local Metadata(%s) does not match requirement", key)
+							return false
+						}
+					} else {
+						log.Debugf("Local Metadata(%s) does not match requirement", key)
+						return false
+					}
+				} else if index := strings.Index(v, ">="); strings.Contains(v, ">=") && (index == 0) {
+					need, err1 := strconv.Atoi(v[2:])
+					have, err2 := strconv.Atoi(local)
+					if (err1 == nil && err2 == nil) {
+						if have >= need {
+							log.Debugf("Local Metadata(%s) meets requirement", key)
+							continue
+						} else {
+							log.Debugf("Local Metadata(%s) does not match requirement", key)
+							return false
+						}
+					} else {
+						log.Debugf("Local Metadata(%s) does not match requirement", key)
+						return false
+					}
+				} else if index := strings.Index(v, ">"); strings.Contains(v, ">") && (index == 0) {
+					need, err1 := strconv.Atoi(v[1:])
+					have, err2 := strconv.Atoi(local)
+					if (err1 == nil && err2 == nil) {
+						if have > need {
+							log.Debugf("Local Metadata(%s) meets requirement", key)
+							continue
+						} else {
+							log.Debugf("Local Metadata(%s) does not match requirement", key)
+							return false
+						}
+					} else {
+						log.Debugf("Local Metadata(%s) does not match requirement", key)
+						return false
+					}
+				} else if index := strings.Index(v, "<"); strings.Contains(v, "<") && (index == 0) {
+					need, err1 := strconv.Atoi(v[1:])
+					have, err2 := strconv.Atoi(local)
+					if (err1 == nil && err2 == nil) {
+						if have < need {
+							log.Debugf("Local Metadata(%s) meets requirement", key)
+							continue
+						} else {
+							log.Debugf("Local Metadata(%s) does not match requirement", key)
+							return false
+						}
+					} else {
+						log.Debugf("Local Metadata(%s) does not match requirement", key)
+						return false
+					}
+				} else if index := strings.Index(v, "!="); strings.Contains(v, "!=") && (index == 0) {
+					if (v[2:] != local) {
+						log.Debugf("Local Metadata(%s) meets requirement", key)
+						continue
+					} else {
+						log.Debugf("Local Metadata(%s) does not match requirement", key)
+						return false
+					}
+				} else {
+					log.Debugf("Local Metadata(%s) does not match requirement", key)
+					return false
+				}
+			}
 		}
 	}

########################## uptime ########################## 1128

diff --git a/agent/unit_state_test.go b/agent/unit_state_test.go
index a999175..0b6b496 100644
--- a/agent/unit_state_test.go
+++ b/agent/unit_state_test.go
@@ -728,7 +728,7 @@ func TestMarshalJSON(t *testing.T) {
 	if err != nil {
 		t.Fatalf("unexpected error marshalling: %v", err)
 	}
-	want = `{"Cache":{"bar.service":{"LoadState":"","ActiveState":"inactive","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"bar.service"},"foo.service":{"LoadState":"","ActiveState":"active","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"foo.service"}},"ToPublish":{"woof.service":{"LoadState":"","ActiveState":"active","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"woof.service"}}}`
+	want = `{"Cache":{"bar.service":{"LoadState":"","ActiveState":"inactive","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"bar.service","ActiveEnterTimestamp":0},"foo.service":{"LoadState":"","ActiveState":"active","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"foo.service","ActiveEnterTimestamp":0}},"ToPublish":{"woof.service":{"LoadState":"","ActiveState":"active","SubState":"","MachineID":"asdf","UnitHash":"","UnitName":"woof.service","ActiveEnterTimestamp":0}}}`
 	if string(got) != want {
 		t.Fatalf("Bad JSON representation: got\n%s\n\nwant\n%s", string(got), want)
 	}
diff --git a/fleetctl/list_units.go b/fleetctl/list_units.go
index 28f682a..967b5af 100644
--- a/fleetctl/list_units.go
+++ b/fleetctl/list_units.go
@@ -18,13 +18,14 @@ import (
 	"fmt"
 	"sort"
 	"strings"
+        "time"
 
 	"github.com/coreos/fleet/machine"
 	"github.com/coreos/fleet/schema"
 )
 
 const (
-	defaultListUnitsFields = "unit,machine,active,sub"
+	defaultListUnitsFields = "unit,machine,active,sub,uptime"
 )
 
 var (
@@ -90,6 +91,14 @@ Or, choose the columns to display:
 			}
 			return us.Hash
 		},
+                "uptime": func(us *schema.UnitState, full bool) string {
+                        if us == nil || us.SystemdActiveState != "active"{
+                                return "-"
+                        }                      
+                        tm := time.Unix(0, int64(us.SystemdActiveEnterTimestamp)*1000)
+                        duration := time.Now().Sub(tm)
+                        return fmt.Sprintf("%s, Since %ss", tm.Format("2006-01-02 03:04:05 PM"), strings.Split(duration.String(),".")[0])
+                },
 	}
 )
 
diff --git a/registry/unit_state.go b/registry/unit_state.go
index 5b67d19..6610479 100644
--- a/registry/unit_state.go
+++ b/registry/unit_state.go
@@ -195,6 +195,7 @@ type unitStateModel struct {
 	SubState     string                `json:"subState"`
 	MachineState *machine.MachineState `json:"machineState"`
 	UnitHash     string                `json:"unitHash"`
+        ActiveEnterTimestamp uint64         `json:"ActiveEnterTimestamp"`
 }
 
 func modelToUnitState(usm *unitStateModel, name string) *unit.UnitState {
@@ -208,6 +209,7 @@ func modelToUnitState(usm *unitStateModel, name string) *unit.UnitState {
 		SubState:    usm.SubState,
 		UnitHash:    usm.UnitHash,
 		UnitName:    name,
+                ActiveEnterTimestamp: usm.ActiveEnterTimestamp,
 	}
 
 	if usm.MachineState != nil {
@@ -233,6 +235,7 @@ func unitStateToModel(us *unit.UnitState) *unitStateModel {
 		ActiveState: us.ActiveState,
 		SubState:    us.SubState,
 		UnitHash:    us.UnitHash,
+                ActiveEnterTimestamp: us.ActiveEnterTimestamp,
 	}
 
 	if us.MachineID != "" {
diff --git a/registry/unit_state_test.go b/registry/unit_state_test.go
index 6c22c6e..ced07fb 100644
--- a/registry/unit_state_test.go
+++ b/registry/unit_state_test.go
@@ -101,7 +101,7 @@ func TestSaveUnitState(t *testing.T) {
 	r := &EtcdRegistry{kAPI: e, keyPrefix: "/fleet/"}
 	j := "foo.service"
 	mID := "mymachine"
-	us := unit.NewUnitState("abc", "def", "ghi", mID)
+	us := unit.NewUnitState("abc", "def", "ghi", mID, 1234567890)
 
 	// Saving nil unit state should fail
 	r.SaveUnitState(j, nil, time.Second)
@@ -123,7 +123,7 @@ func TestSaveUnitState(t *testing.T) {
 	us.UnitHash = "quickbrownfox"
 	r.SaveUnitState(j, us, time.Second)
 
-	json := `{"loadState":"abc","activeState":"def","subState":"ghi","machineState":{"ID":"mymachine","PublicIP":"","Metadata":null,"Version":""},"unitHash":"quickbrownfox"}`
+	json := `{"loadState":"abc","activeState":"def","subState":"ghi","machineState":{"ID":"mymachine","PublicIP":"","Metadata":null,"Version":""},"unitHash":"quickbrownfox","ActiveEnterTimestamp":1234567890}`
 	p1 := "/fleet/state/foo.service"
 	p2 := "/fleet/states/foo.service/mymachine"
 	want := []action{
@@ -204,6 +204,7 @@ func TestUnitStateToModel(t *testing.T) {
 				MachineID:   "",
 				UnitHash:    "",
 				UnitName:    "name",
+                                ActiveEnterTimestamp: 0,
 			},
 			want: &unitStateModel{
 				LoadState:    "foo",
@@ -211,6 +212,7 @@ func TestUnitStateToModel(t *testing.T) {
 				SubState:     "baz",
 				MachineState: nil,
 				UnitHash:     "",
+                                ActiveEnterTimestamp: 0,
 			},
 		},
 		{
@@ -222,6 +224,7 @@ func TestUnitStateToModel(t *testing.T) {
 				MachineID:   "",
 				UnitHash:    "heh",
 				UnitName:    "name",
+                                ActiveEnterTimestamp: 1234567890,
 			},
 			want: &unitStateModel{
 				LoadState:    "foo",
@@ -229,6 +232,7 @@ func TestUnitStateToModel(t *testing.T) {
 				SubState:     "baz",
 				MachineState: nil,
 				UnitHash:     "heh",
+                                ActiveEnterTimestamp: 1234567890,
 			},
 		},
 		{
@@ -239,6 +243,7 @@ func TestUnitStateToModel(t *testing.T) {
 				MachineID:   "woof",
 				UnitHash:    "miaow",
 				UnitName:    "name",
+                                ActiveEnterTimestamp: 54321,
 			},
 			want: &unitStateModel{
 				LoadState:    "foo",
@@ -246,6 +251,7 @@ func TestUnitStateToModel(t *testing.T) {
 				SubState:     "baz",
 				MachineState: &machine.MachineState{ID: "woof"},
 				UnitHash:     "miaow",
+                                ActiveEnterTimestamp: 54321,
 			},
 		},
 	} {
@@ -266,7 +272,7 @@ func TestModelToUnitState(t *testing.T) {
 			want: nil,
 		},
 		{
-			in: &unitStateModel{"foo", "bar", "baz", nil, ""},
+			in: &unitStateModel{"foo", "bar", "baz", nil, "", 1234567890},
 			want: &unit.UnitState{
 				LoadState:   "foo",
 				ActiveState: "bar",
@@ -274,10 +280,11 @@ func TestModelToUnitState(t *testing.T) {
 				MachineID:   "",
 				UnitHash:    "",
 				UnitName:    "name",
+                                ActiveEnterTimestamp: 1234567890,
 			},
 		},
 		{
-			in: &unitStateModel{"z", "x", "y", &machine.MachineState{ID: "abcd"}, ""},
+			in: &unitStateModel{"z", "x", "y", &machine.MachineState{ID: "abcd"}, "", 987654321},
 			want: &unit.UnitState{
 				LoadState:   "z",
 				ActiveState: "x",
@@ -285,6 +292,7 @@ func TestModelToUnitState(t *testing.T) {
 				MachineID:   "abcd",
 				UnitHash:    "",
 				UnitName:    "name",
+                                ActiveEnterTimestamp: 987654321,
 			},
 		},
 	} {
diff --git a/schema/mapper.go b/schema/mapper.go
index e4dd05c..1017b61 100644
--- a/schema/mapper.go
+++ b/schema/mapper.go
@@ -121,6 +121,7 @@ func MapUnitStateToSchemaUnitState(entity *unit.UnitState) *UnitState {
 		SystemdLoadState:   entity.LoadState,
 		SystemdActiveState: entity.ActiveState,
 		SystemdSubState:    entity.SubState,
+                SystemdActiveEnterTimestamp: entity.ActiveEnterTimestamp,
 	}
 
 	return &us
@@ -136,6 +137,7 @@ func MapSchemaUnitStatesToUnitStates(entities []*UnitState) []*unit.UnitState {
 			LoadState:   e.SystemdLoadState,
 			ActiveState: e.SystemdActiveState,
 			SubState:    e.SystemdSubState,
+                        ActiveEnterTimestamp: e.SystemdActiveEnterTimestamp,
 		}
 	}
 
diff --git a/schema/v1-gen.go b/schema/v1-gen.go
index 24a215e..e370a1e 100644
--- a/schema/v1-gen.go
+++ b/schema/v1-gen.go
@@ -141,6 +141,7 @@ type UnitState struct {
 	SystemdLoadState string `json:"systemdLoadState,omitempty"`
 
 	SystemdSubState string `json:"systemdSubState,omitempty"`
+        SystemdActiveEnterTimestamp uint64 `json:"systemdActiveEnterTimestamp,omitempty"`
 }
 
 type UnitStatePage struct {
diff --git a/systemd/manager.go b/systemd/manager.go
index d01b7b3..5cb8a6a 100644
--- a/systemd/manager.go
+++ b/systemd/manager.go
@@ -238,6 +238,17 @@ func (m *systemdUnitManager) GetUnitStates(filter pkg.Set) (map[string]*unit.Uni
 		states[name] = us
 	}
 
+        // add Active enter time to UnitState
+        for name, us := range states {
+                prop, err := m.systemd.GetUnitProperty(name, "ActiveEnterTimestamp")
+                if err != nil {
+                        return nil, err
+                }
+                
+                us.ActiveEnterTimestamp = prop.Value.Value().(uint64)
+                states[name] = us
+        }
+
 	return states, nil
 }
 
diff --git a/unit/fake.go b/unit/fake.go
index bc82352..fd450e7 100644
--- a/unit/fake.go
+++ b/unit/fake.go
@@ -83,7 +83,7 @@ func (fum *FakeUnitManager) GetUnitStates(filter pkg.Set) (map[string]*UnitState
 	states := make(map[string]*UnitState)
 	for _, name := range filter.Values() {
 		if _, ok := fum.u[name]; ok {
-			states[name] = &UnitState{"loaded", "active", "running", "", "", name}
+			states[name] = &UnitState{"loaded", "active", "running", "", "", name, 0}
 		}
 	}
 
diff --git a/unit/fake_test.go b/unit/fake_test.go
index 3b616f2..2df16cd 100644
--- a/unit/fake_test.go
+++ b/unit/fake_test.go
@@ -60,7 +60,7 @@ func TestFakeUnitManagerLoadUnload(t *testing.T) {
 		t.Fatalf("Expected non-nil UnitState")
 	}
 
-	eus := NewUnitState("loaded", "active", "running", "")
+	eus := NewUnitState("loaded", "active", "running", "", 0)
 	if !reflect.DeepEqual(*us, *eus) {
 		t.Fatalf("Expected UnitState %v, got %v", eus, *us)
 	}
diff --git a/unit/generator_test.go b/unit/generator_test.go
index bd3fe80..b99ccfd 100644
--- a/unit/generator_test.go
+++ b/unit/generator_test.go
@@ -49,7 +49,7 @@ func TestUnitStateGeneratorSubscribeLifecycle(t *testing.T) {
 
 	// subscribed to foo.service so we should get a heartbeat
 	expect := []UnitStateHeartbeat{
-		UnitStateHeartbeat{Name: "foo.service", State: &UnitState{"loaded", "active", "running", "", "", "foo.service"}},
+		UnitStateHeartbeat{Name: "foo.service", State: &UnitState{"loaded", "active", "running", "", "", "foo.service", 0}},
 	}
 	assertGenerateUnitStateHeartbeats(t, um, gen, expect)
 
diff --git a/unit/unit.go b/unit/unit.go
index 4f9be39..24b5dbe 100644
--- a/unit/unit.go
+++ b/unit/unit.go
@@ -177,14 +177,16 @@ type UnitState struct {
 	MachineID   string
 	UnitHash    string
 	UnitName    string
+        ActiveEnterTimestamp uint64
 }
 
-func NewUnitState(loadState, activeState, subState, mID string) *UnitState {
+func NewUnitState(loadState, activeState, subState, mID string, activeEnterTimestamp uint64) *UnitState {
 	return &UnitState{
 		LoadState:   loadState,
 		ActiveState: activeState,
 		SubState:    subState,
 		MachineID:   mID,
+                ActiveEnterTimestamp: activeEnterTimestamp,
 	}
 }
 
diff --git a/unit/unit_test.go b/unit/unit_test.go
index d384467..7d73d95 100644
--- a/unit/unit_test.go
+++ b/unit/unit_test.go
@@ -97,9 +97,10 @@ func TestNewUnitState(t *testing.T) {
 		ActiveState: "as",
 		SubState:    "ss",
 		MachineID:   "id",
+                ActiveEnterTimestamp: 1234567890,
 	}
 
-	got := NewUnitState("ls", "as", "ss", "id")
+	got := NewUnitState("ls", "as", "ss", "id", 1234567890)
 	if !reflect.DeepEqual(got, want) {
 		t.Fatalf("NewUnitState did not create a correct UnitState: got %s, want %s", got, want)
 	}


########################## Wood ##########################

export GOROOT=/home/lightos/golang/go
export PATH=$PATH:$GOROOT/bin
Wood 953 



    1293
    agent/reconcile.go
    registry/job.go
    
    1270
    agent/reconcile.go    
    agent/state_test.go
    api/units_test.go

    1271
    job/job.go
    job/job_test.go



diff --git a/fleetctl/fleetctl.go b/fleetctl/fleetctl.go
index 702918b..e39294a 100644
--- a/fleetctl/fleetctl.go
+++ b/fleetctl/fleetctl.go
@@ -167,6 +167,7 @@ func init() {
 		cmdListUnitFiles,
 		cmdListUnits,
 		cmdLoadUnits,
+		cmdOverwriteUnit,
 		cmdSSH,
 		cmdStartUnit,
 		cmdStatusUnits,
@@ -175,7 +176,6 @@ func init() {
 		cmdUnloadUnit,
 		cmdVerifyUnit,
 		cmdVersion,
-		cmdOverwriteUnit,
 	}
 }
 
diff --git a/fleetctl/load.go b/fleetctl/load.go
index 497f860..55c76e4 100644
--- a/fleetctl/load.go
+++ b/fleetctl/load.go
@@ -45,12 +45,7 @@ func init() {
 	cmdLoadUnits.Flags.BoolVar(&sharedFlags.NoBlock, "no-block", false, "Do not wait until the jobs have been loaded before exiting. Always the case for global units.")
 }
 
-func runLoadUnits(args []string) (exit int) {
-	if err := lazyCreateUnits(args); err != nil {
-		stderr("Error creating units: %v", err)
-		return 1
-	}
-
+func loadUnits(args []string) (exit init) {
 	triggered, err := lazyLoadUnits(args)
 	if err != nil {
 		stderr("Error loading units: %v", err)
@@ -80,3 +75,11 @@ func runLoadUnits(args []string) (exit int) {
 
 	return
 }
+
+func runLoadUnits(args []string) (exit int) {
+	if err := lazyCreateUnits(args); err != nil {
+		stderr("Error creating units: %v", err)
+		return 1
+	}
+	return loadUnits(args)
+}
diff --git a/fleetctl/overwrite.go b/fleetctl/overwrite.go
index 4150481..9f2a676 100644
--- a/fleetctl/overwrite.go
+++ b/fleetctl/overwrite.go
@@ -9,14 +9,27 @@ import (
 var cmdOverwriteUnit = &Command{
 	Name:    "overwrite",
 	Summary: "Overwrite one or more units in the cluster",
-	Usage:   "UNIT...",
-	Description: `Overwrite one or more running or submitted units from the cluster.
+	Usage:   "[--no-block|--block-attempts=N] UNIT...",
+	Description: `Overwrite stops (if needed), unloads (if needed), and destroys, then submits, loads (if needed) and starts (if needed) the unit after replacing the unit file in the registry with that found on the local filesystem.
 
-Act as a procedure of stop-->destroy-->commit-->start(if needed) on unit(s)`,
+Act on unit(s) as a procedure of:
+	stop(if needed)-->unload(if needed)-->destroy-->submit-->load(if needed)-->start(if needed)
+
+Overwrite a single unit:
+	fleetctl overwrite foo.service
+
+Overwrite a directory of units with glob matching:
+	fleetctl overwrite myservice/*`,
 
 	Run: runOverwriteUnits,
 }
 
+func init() {
+	cmdLoadUnits.Flags.IntVar(&sharedFlags.BlockAttempts, "block-attempts", 0, "Wait until the jobs are loaded, performing up to N attempts before giving up. A value of 0 indicates no limit. Does not apply to global units.")
+	cmdLoadUnits.Flags.BoolVar(&sharedFlags.NoBlock, "no-block", false, "Do not wait until the jobs have been loaded before exiting. Always the case for global units.")
+}
+
+
 func runOverwriteUnits(args []string) (exit int) {
 	for _, v := range args {
 		v = maybeAppendDefaultUnitType(v)
@@ -24,29 +37,29 @@ func runOverwriteUnits(args []string) (exit int) {
 
 		u, err := cAPI.Unit(name)
 		if err != nil {
-			stderr("error retrieving Unit(%s) from Registry: %v", name, err)
+			stderr("Error retrieving unit(%s) from registry: %v", name, err)
 			return 1
 		}
 		if u == nil {
-			stdout("Unit(%s) in can not be found in Registry, nothing to do with it", name)
+			stdout("Unit(%s) can not be found in registry, nothing to do with it.", name)
 			continue
 		}
 
 		hash_mismatch := warnOnDifferentLocalUnit(v, u)
 		if hash_mismatch == false {
-			stdout("Nothing different between Unit(%s) in registory and local unit file, just skip", name)
+			stdout("The unit(%s) has nothing different between in registory and local unit file, ignoring.", name)
 			continue
 		}
 
 		stopping := make([]string, 0)
 		stopping = append(stopping, u.Name)
 		if job.JobState(u.CurrentState) == job.JobStateLaunched || suToGlobal(*u) {
-			stdout("Stop Unit(%s) first, so that we can overwrite unit file", u.Name)
+			stdout("Stop unit(%s) first, so that we can overwrite unit file", u.Name)
 			cAPI.SetUnitTargetState(u.Name, string(job.JobStateLoaded))
 			errchan := waitForUnitStates(stopping, job.JobStateLoaded, 0, os.Stdout)
 			for err := range errchan {
 				stderr("Error waiting for units: %v", err)
-				exit = 1
+				return 1
 			}
 		}
 
@@ -59,54 +72,25 @@ func runOverwriteUnits(args []string) (exit int) {
 		recreating = append(recreating, v)
 		stdout("Recreating unit(%s)", v)
 		if err := lazyCreateUnits(recreating); err != nil {
-			stderr("Error creating units: %v", err)
-			return
+			stderr("Error creating unit: %v", err)
+			return 1
 		}
 		if job.JobState(u.CurrentState) == job.JobStateInactive {
 			continue
 		} else if job.JobState(u.CurrentState) == job.JobStateLoaded {
-			stdout("Rescheduling unit(%s)", v)
-			triggered, err := lazyLoadUnits(recreating)
-			if err != nil {
-				stderr("Error loading units: %v", err)
+			stdout("Reloading unit(%s)", v)
+			err := loadUnit(recreating)
+			if (err != nil) {
+				stderr("Error reloading unit: %v", err)
 				return 1
 			}
-
-			var loading []string
-			for _, u := range triggered {
-				if suToGlobal(*u) {
-					stdout("Triggered global unit %s load", u.Name)
-				} else {
-					loading = append(loading, u.Name)
-				}
-			}
-
-			errchan := waitForUnitStates(loading, job.JobStateLoaded, sharedFlags.BlockAttempts, os.Stdout)
-			for err := range errchan {
-				stderr("Error waiting for units: %v", err)
-			}
 		} else if job.JobState(u.CurrentState) == job.JobStateLaunched {
 			stdout("Restarting unit(%s)", v)
-			triggered, err := lazyStartUnits(recreating)
-			if err != nil {
-				stderr("Error starting units: %v", err)
+			err := startUnits(recreating)
+			if (err != nil) {
+				stderr("Error restarting unit: %v", err)
 				return 1
 			}
-
-			var starting []string
-			for _, u := range triggered {
-				if suToGlobal(*u) {
-					stdout("Triggered global unit %s start", u.Name)
-				} else {
-					starting = append(starting, u.Name)
-				}
-			}
-
-			errchan := waitForUnitStates(starting, job.JobStateLaunched, sharedFlags.BlockAttempts, os.Stdout)
-			for err := range errchan {
-				stderr("Error waiting for units: %v", err)
-				exit = 1
-			}
 		}
 
 	}
diff --git a/fleetctl/start.go b/fleetctl/start.go
index 77ea0d3..7dc29f2 100644
--- a/fleetctl/start.go
+++ b/fleetctl/start.go
@@ -53,12 +53,7 @@ func init() {
 	cmdStartUnit.Flags.BoolVar(&sharedFlags.NoBlock, "no-block", false, "Do not wait until the units have launched before exiting. Always the case for global units.")
 }
 
-func runStartUnit(args []string) (exit int) {
-	if err := lazyCreateUnits(args); err != nil {
-		stderr("Error creating units: %v", err)
-		return 1
-	}
-
+func startUnit(args []string) (exit int) {
 	triggered, err := lazyStartUnits(args)
 	if err != nil {
 		stderr("Error starting units: %v", err)
@@ -88,3 +83,11 @@ func runStartUnit(args []string) (exit int) {
 
 	return
 }
+
+func runStartUnit(args []string) (exit int) {
+	if err := lazyCreateUnits(args); err != nil {
+		stderr("Error creating units: %v", err)
+		return 1
+	}
+	return startUnit(args)
+}
