``` bash 
(echo "NAMESPACE                PODS     CPU(m)      MEM(Mi)"; 
 kubectl top pod -A --no-headers | awk '
function cpu_m(x){ if (x ~ /m$/) {sub(/m$/,"",x); return x+0} return (x+0)*1000 }
function mem_mi(x){ if (x ~ /Ki$/){sub(/Ki$/,"",x); return (x+0)/1024} if (x ~ /Mi$/){sub(/Mi$/,"",x); return x+0} if (x ~ /Gi$/){sub(/Gi$/,"",x); return (x+0)*1024} if (x ~ /Ti$/){sub(/Ti$/,"",x); return (x+0)*1024*1024} return x+0 }
{ ns=$1; cpu[ns]+=cpu_m($3); mem[ns]+=mem_mi($4); pods[ns]++ }
END{ for (n in pods) printf "%-22s %6d %10.0f %12.0f\n", n,pods[n],cpu[n],mem[n] }
' | sort -k3,3nr ) | head -n 20


```