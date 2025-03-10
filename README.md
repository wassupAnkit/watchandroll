# watchandroll
combine best of both watchtower and docker rollout
package main

import (
	"context"
	"log"
	"time"

	"github.com/docker/docker/api/types"
	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/client"
)

// checkForUpdate is a stub function. Replace its logic with a real check against your Docker registry.
func checkForUpdate(currentImage string) (bool, string) {
	// Dummy implementation: no updates found.
	// In practice, you’d check the registry for a newer digest or version.
	return false, ""
}

// deployUpdatedContainer replicates the container with new image parameters and performs a rolling update.
func deployUpdatedContainer(ctx context.Context, cli *client.Client, containerID string, newImage string) error {
	// Retrieve the configuration of the current container.
	contJSON, err := cli.ContainerInspect(ctx, containerID)
	if err != nil {
		return err
	}

	// Prepare the configuration for a new container instance.
	newConfig := contJSON.Config
	newConfig.Image = newImage
	newHostConfig := contJSON.HostConfig

	// Optionally, override settings such as container name.
	// Create the new container.
	resp, err := cli.ContainerCreate(ctx, newConfig, newHostConfig, nil, nil, "")
	if err != nil {
		return err
	}
	newContainerID := resp.ID

	// Start the new container.
	if err := cli.ContainerStart(ctx, newContainerID, types.ContainerStartOptions{}); err != nil {
		return err
	}
	log.Printf("Started new container %s with image %s", newContainerID, newImage)

	// Insert health-check logic here. For example:
	// - Poll the container status until it passes an HTTP health endpoint check.
	// - Or wait a predefined period ensuring sufficient startup time.
	// For now, we simply sleep for a short interval.
	time.Sleep(15 * time.Second)

	// Once confirmed that the new container is healthy, gracefully stop the old one.
	timeout := 10 * time.Second
	if err := cli.ContainerStop(ctx, containerID, &timeout); err != nil {
		return err
	}
	log.Printf("Stopped old container %s", containerID)

	// Optionally, remove the old container.
	if err := cli.ContainerRemove(ctx, containerID, types.ContainerRemoveOptions{Force: true}); err != nil {
		return err
	}
	log.Printf("Removed old container %s", containerID)

	return nil
}

func main() {
	ctx := context.Background()
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		log.Fatalf("Error creating Docker client: %v", err)
	}

	// Use a ticker to perform periodic update checks (adjust the interval as needed).
	ticker := time.NewTicker(5 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// List currently running containers.
			containers, err := cli.ContainerList(ctx, types.ContainerListOptions{})
			if err != nil {
				log.Printf("Error listing containers: %v", err)
				continue
			}

			// Evaluate each container.
			for _, cont := range containers {
				// Check if an updated image is available.
				updateAvailable, newImage := checkForUpdate(cont.Image)
				if updateAvailable {
					log.Printf("Update found for container %s: preparing to update to image %s", cont.ID, newImage)
					// Perform a rolling update.
					if err := deployUpdatedContainer(ctx, cli, cont.ID, newImage); err != nil {
						log.Printf("Error updating container %s: %v", cont.ID, err)
					}
				}
			}
		}
	}
}
