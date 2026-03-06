# k8s运行windows系统

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows
  namespace: windows
  labels:
    app: windows
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: windows
  template:
    metadata:
      labels:
        app: windows
    spec:
      runtimeClassName: nvidia
#      hostNetwork: true
      containers:
        - name: windows
          image: dockurr/windows:4.10
          env:
#            - name: GPU
#              value: "Y"
            - name: VERSION
              value: "11"
            - name: DISK_SIZE
              value: "200G"
            - name: DISK2_SIZE
              value: "100G"
            - name: RAM_SIZE
              value: "16G"
            - name: CPU_CORES
              value: "4"
            - name: USERNAME
              value: "xiaowu"
            - name: PASSWORD
              value: "xiaowu"
            - name: LANGUAGE
              value: "Chinese"
            - name: DHCP
              value: "Y"
            - name: ARGUMENTS
              value: '-device vfio-pci,host=01:00.0,multifunction=on'
#               -cpu host,-hypervisor,+kvm_pv_unhalt,+kvm_pv_eoi,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_reset,hv_vpindex,hv_runtime,hv_relaxed,kvm=off,hv_vendor_id=intel'
          ports:
            - containerPort: 8006
            - containerPort: 3389
          volumeMounts:
            - mountPath: /storage
              name: windows-storage
            - mountPath: /storage2
              name: windows-share
            - mountPath: /dev/kvm
              name: kvm-device
            - mountPath: /dev/net/tun
              name: tun-device
#            - mountPath: /custom.iso
#              name: iso
          securityContext:
            privileged: true
          resources:
            limits:
              nvidia.com/gpu: 1
      terminationGracePeriodSeconds: 120
      volumes:
        - name: windows-storage
          hostPath:
            path: /data/SSD/app/windows-storage
        - name: windows-share
          hostPath:
            path: /opt/windows-share
        - name: kvm-device
          hostPath:
            path: /dev/kvm
        - name: tun-device
          hostPath:
            path: /dev/net/tun
            type: CharDevice
#        - name: iso
#          hostPath:
#            path: /data/SSD/ISO/bwcx_Windows11_24H2_26100.2161_X64_2024.10.27.iso
#            type: File
      nodeSelector:
        nvidia.com/gpu.product: NVIDIA-GeForce-GTX-1060
---
apiVersion: v1
kind: Service
metadata:
  name: windows-service
  namespace: windows
spec:
  type: NodePort
  selector:
    app: windows
  ports:
    - name: http
      port: 8006
      nodePort: 8006
      targetPort: 8006
    - name: rdp-tcp
      port: 3389
      targetPort: 3389
      nodePort: 3391
      protocol: TCP
    - name: rdp-udp
      port: 3389
      targetPort: 3389
      nodePort: 3391
      protocol: UDP
```

