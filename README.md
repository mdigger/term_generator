# Embedded migration generator fo [tern](https://github.com/jackc/tern/tree/master/migrate)

```golang
//go:generate go run generator.go

package db

import (
	"context"
	"errors"
	"time"

	"github.com/jackc/pgx/v4/pgxpool"
	"github.com/jackc/tern/migrate"
	log "github.com/sirupsen/logrus"
)

var (
	// VersionTable contains the name of the table with the current version of the schema
	VersionTable = "public.schema_version"
	// preloaded data for migration (generated)
	migrations []*migrate.Migration
)

// Migrate checks the database schema and updates it if necessary.
func Migrate(pool *pgxpool.Pool, timeout time.Duration) error {
	if migrations == nil {
		return errors.New("migration scripts not defined")
	}
	var (
		ctx    = context.Background()
		cancel context.CancelFunc
	)
	if timeout > 0 {
		ctx, cancel = context.WithTimeout(ctx, timeout)
		defer cancel()
	}
	pconn, err := pool.Acquire(ctx)
	if err != nil {
		return err
	}
	defer pconn.Release()
	migrator, err := migrate.NewMigrator(ctx, pconn.Conn(), VersionTable)
	if err != nil {
		return err
	}
	// substitute pre-parsed migrations
	migrator.Migrations = migrations
	if err = migrator.Migrate(ctx); err != nil {
		return err
	}
	ver, err := migrator.GetCurrentVersion(ctx)
	if err != nil {
		return err
	}
	log.WithField("ver", ver).Info("db initialized")
	return nil
}
```