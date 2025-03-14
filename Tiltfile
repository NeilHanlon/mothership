#
# Services
# 

load('ext://cert_manager', 'deploy_cert_manager')
load('ext://configmap', 'configmap_create')
load('ext://helm_remote', 'helm_remote')


dbPassword = '43254SSfsdAA32ds3232sds'

helm_remote(
  'postgresql',
  repo_url='https://charts.bitnami.com/bitnami',
  set=["auth.database=mothership","auth.postgresPassword=" + dbPassword],
  namespace='mothership',
)
k8s_resource('postgresql', port_forwards=[5432])

helm_remote(
  'valkey',
  repo_url='oci://registry-1.docker.io/bitnamicharts',
  set=["auth.password=" + dbPassword],
  namespace='mothership',
)
k8s_resource('valkey-primary', port_forwards=[6379])

k8s_yaml('k8s/dev/dex.yaml')
configmap_create('dex-config', from_file=['dex.yaml=./dev/dex.yaml'])
k8s_resource('dev-dex', port_forwards=[5556])

k8s_yaml('k8s/dev/temporal.yaml')
k8s_resource('dev-temporal', port_forwards=[7233,8233])

deploy_cert_manager()

#
# Secrets
#

load('ext://secret', 'secret_create_generic', 'secret_from_dict')

k8s_yaml(secret_from_dict("db", namespace='mothership', inputs={'uri': "postgres://postgres:" + dbPassword + "@postgresql/mothership?sslmode=disable"}))

k8s_yaml(secret_from_dict("gh", namespace='mothership', inputs={'client_id': '', 'client_secret': ''}))
k8s_yaml(secret_from_dict("csrf", namespace='mothership', inputs={'secret': dbPassword+dbPassword}))
k8s_yaml(secret_from_dict("redis", namespace='mothership', inputs={'password': dbPassword}))

# Load all Kubernetes manifests
k8s_yaml([
    "k8s/admin-api.yaml",
    "k8s/api.yaml",
    "k8s/namespace.yaml",
    "k8s/serviceaccount.yaml",
    "k8s/ui.yaml",
    "k8s/worker.yaml",
])

objects = read_yaml_stream('k8s/cert.yaml')
for o in objects:
  o['spec']['dnsNames'] = ['mothership.local']
k8s_yaml(encode_yaml_stream(objects))

# Build images using `ko`
load('ext://ko', 'ko_build')

ko_build("github.com/openela/mothership/cmd/mship_server", "./cmd/mship_server")
ko_build("github.com/openela/mothership/cmd/mship_admin_server", "./cmd/mship_admin_server")
ko_build("github.com/openela/mothership/cmd/mship_worker_server", "./cmd/mship_worker_server")

# Assign local port-forwards for easier access
k8s_resource("mothership-ui-deployment", port_forwards=9111, resource_deps=['mothership-api-deployment', 'mothership-admin-api-deployment'])
k8s_resource("mothership-api-deployment", port_forwards=6677)
k8s_resource("mothership-admin-api-deployment", port_forwards=6687)
k8s_resource("mothership-worker-deployment", port_forwards=9114)
