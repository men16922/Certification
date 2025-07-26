# 4. JSONPATH 쿼리

# Q9

Use JSON PATH query to retrieve the osImages of all the nodes and store it in a file
all-nodes-os-info.txt at root location.

Note: The osImage are under the nodeInfo section under status of each node.

Answer
```bash
kubectl get no -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}' \
> all-nodes-os.info.txt
cat all-nodes-os.info.txt
```