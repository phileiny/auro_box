1.Swat team 雲平台小組
  代理 代工 技術(品牌) 模組化 交叉變化

order_task_update_signal

export PGPASSWORD=$(sudo kubectl get secret tsdb-credentials -n infrastructure -o jsonpath='{.data.PATRONI_SUPERUSER_PASSWORD}' | base64 -d)

export PGPASSWORD=$(kubectl get secret tsdb-credentials -n infrastructure -o jsonpath='{.data.PATRONI_SUPERUSER_PASSWORD}' | base64 -d)


XozIhIj6IjRDbDxXz4Qug4zAd4rNI0qy

export PGPASSWORD='XozIhIj6IjRDbDxXz4Qug4zAd4rNI0qy'

{"PATRONI_REPLICATION_PASSWORD":"b3RuN3lPQnhBOEVvaWEyS2hxWGRpU1pKRlNGdnNtYjI=","PATRONI_SUPERUSER_PASSWORD":"WG96SWhJajZJalJEYkR4WHo0UXVnNHpBZDRyTkkwcXk=","PATRONI_admin_PASSWORD":"TjVmWHltVXlidnJNTHNBRWtSSzIwMkd0bHViRUI4bXg="}


export PGPASSWORD=$(kubectl get secret tsdb-credentials -n infrastructure -o jsonpath='{.data.PATRONI_admin_PASSWORD}' | base64 -d)

admin N5fXymUybvrMLsAEkRK202GtlubEB8mx


app=tsdb-timescaledb,apps.kubernetes.io/pod-index=0,cluster-name=tsdb,controller-revision-hash=tsdb-timescaledb-5bdd684c65,release=tsdb,role=master,statefulset.kubernetes.io/pod-name=tsdb-timescaledb-0

export PGPASSWORD= 'N5fXymUybvrMLsAEkRK202GtlubEB8mx'
psql -U admin -h 43.213.110.250 -p 30432 -d app_aiyo -c "SELECT now();"